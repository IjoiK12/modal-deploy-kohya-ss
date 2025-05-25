# Run Kohya_SS GUI on Modal - Cloud-Based LoRA Training

This repository provides a working configuration to run the popular Kohya_SS GUI (by bmaltais) on the [Modal](https://modal.com) serverless platform. This setup allows you to leverage powerful cloud GPUs (A10G, A100, H100, etc.) for training LoRA models (and other Dreambooth-style training) without complex local installations or CUDA/driver headaches.

This project aims to provide a straightforward path for users 누가to train their models in the cloud using a familiar web interface.

## Features

* **Cloud GPU Access:** Train on various NVIDIA GPUs, paying only for usage.
* **Simplified Setup:** Avoids most local environment configuration issues.
* **Familiar Web UI:** Uses the standard Kohya_SS Gradio interface.
* **Persistent Storage:** Utilizes `modal.Volume` for models, datasets, and outputs, ensuring data persistence between sessions.
* **Controlled Updates:** Includes a mechanism to update the Kohya_SS version when desired.

## Prerequisites

Before you begin, ensure you have the following:

1.  **A Modal Account:** Sign up at [modal.com](https://modal.com). New users often receive free credits.
2.  **Modal Client Installed and Configured:**
    ```bash
    pip install modal-client
    modal token new
    ```
3.  **Python:** Python 3.10 or newer installed locally.
4.  **Git:** For cloning this repository.
5.  **Training Data:** Your base models (e.g., `.safetensors` files) and image datasets prepared for Kohya_SS.

## Setup Instructions

1.  **Clone this Repository:**
    ```bash
    git clone https://github.com/IjoiK12/modal-deploy-kohya-ss.git
    cd modal-deploy-kohya-ss
    ```

2.  **Configure `config.toml`:**
    Create a file named `config.toml` in the root of the cloned repository with the following content, adjusting parameters as needed:

    ```toml
    [modal_settings]
    # allow_concurrent_inputs = 10  # Max concurrent requests for @modal.concurrent
    container_idle_timeout = 600  # Idle time in seconds before container scales down (used for scaledown_window)
    timeout = 7200                # Max container lifetime in seconds (e.g., 2 hours)
    gpu = "A10G"                  # GPU type: "A10G", "T4", "L4", "A100", "H100"

    [kohya_settings]
    port = 8000                   # Port for the web UI inside the container
    ```
    * `gpu`: Choose based on your needs. `A10G` (24GB VRAM) is a good starting point for many tasks. For SDXL or larger batches, consider `A100` (40GB or 80GB) or `H100` (80GB).
    * `container_idle_timeout`: This is used for `scaledown_window`. 600 seconds = 10 minutes. If `min_containers` is 0 (default or not set in `app.py`), the container will stop after this period of inactivity.
    * `timeout`: Maximum duration a container can run. Adjust if you expect very long training sessions.

3.  **Review `app.py`:**
    The provided `app.py` contains the Modal application definition, including the image build process and runtime function. It is configured based on extensive debugging to ensure a stable environment for Kohya_SS.

## Uploading Data (Models & Datasets)

Your models, datasets, and outputs will be stored in persistent Modal Volumes. You need to upload your base models and datasets to these volumes using the `modal volume put` command from your local terminal.

The `app.py` script maps these volumes to paths inside the container:
* `kohya-models` volume is mounted at `/kohya_ss/models/`
* `kohya-dataset` volume is mounted at `/kohya_ss/dataset/` (Note: singular "dataset" in the path as per your `app.py`)
* `kohya-outputs` volume is mounted at `/kohya_ss/outputs/`
* `kohya-configs` volume is mounted at `/kohya_ss/configs/`

**Uploading Base Models:**
   * Volume Name: `kohya-models`
   * Example: If your model `my_sdxl_model.safetensors` is locally at `C:\AI\Models\my_sdxl_model.safetensors`:
        ```bash
        modal volume put kohya-models C:\AI\Models\my_sdxl_model.safetensors /my_sdxl_model.safetensors
        ```
        This makes the model available inside the container at `/kohya_ss/models/my_sdxl_model.safetensors`.

**Uploading Datasets:**
   * Volume Name: `kohya-dataset`
   * Kohya_SS expects a specific directory structure for datasets, typically: `Your_Image_Folder_In_GUI/Repeats_InstanceToken/image.png`.
   * Example: If your processed dataset folder (e.g., `40_mycharacter_style`) is locally at `D:\TrainingData\my_style_project\40_mycharacter_style`:
        ```bash
        modal volume put kohya-dataset D:\TrainingData\my_style_project\40_mycharacter_style /40_mycharacter_style
        ```
        This makes the dataset available inside the container at `/kohya_ss/dataset/40_mycharacter_style/`. When using the Kohya GUI, you would set "Image folder" to `/kohya_ss/dataset/`.

**Verifying Volume Contents:**
   * You can list the contents of your volumes:
        ```bash
        modal volume ls kohya-models -r
        modal volume ls kohya-dataset -r
        ```

## Running the Application

You have two primary ways to run the application:

1.  **Temporary Run (for Development/Testing):**
    ```bash
    modal serve app.py
    ```
    The application will run as long as this command is active in your terminal. Modal will provide a temporary URL to access the GUI. Press `Ctrl+C` to stop.

2.  **Persistent Deployment:**
    ```bash
    modal deploy app.py
    ```
    This deploys the application to Modal, where it will run in the background and be accessible via a persistent URL. You can close your terminal. To update the deployment after code changes, run this command again.

Modal will output the URL (e.g., `https://your_username--kohya-ss-gui-run-kohya-gui-dev.modal.run`) for the web interface.

## Using the Kohya_SS GUI

1.  Open the URL provided by Modal in your web browser.
2.  Navigate to the desired training tab (e.g., LoRA, Dreambooth).
3.  **Crucially, when specifying paths in the GUI, use the paths *inside the container*:**
    * **Pretrained model name or path:** `/kohya_ss/models/your_model_name.safetensors`
    * **Image folder (Dataset directory):** `/kohya_ss/dataset/` (Kohya will then look for your `Repeats_InstanceToken` subfolders inside this path).
    * **Output folder:** `/kohya_ss/outputs/`
    * **Logging folder:** `/kohya_ss/outputs/logs` (or your preference within `/kohya_ss/outputs/`)
    * **LoRA model output name:** (e.g., `my_awesome_lora`)
4.  Configure all other training parameters as desired.
5.  Start the training.

## Downloading Results

Your trained models (LoRA files, etc.) will be saved to the `/kohya_ss/outputs/` directory within the `kohya-outputs` volume. Use `modal volume get` to download them:

```bash
modal volume get kohya-outputs /my_awesome_lora.safetensors C:\LoRAs\my_awesome_lora.safetensors