# flac16-rar-to-mp3-v0
# readme_generator.py
# This script prints the full Markdown content for the README.md file.

README_CONTENT = """# RAR to MP3 Converter for Music Albums

This is an automated script to process a directory of music albums archived in `.rar` format. It extracts FLAC files from each archive, converts them to high-quality V0 MP3s, and organizes the output into a clean directory structure.

The script is designed for Linux environments, specifically Ubuntu Server 24.04, and uses standard command-line tools for maximum efficiency and reliability.

## Features

-   **Batch Processing**: Automatically processes every `.rar` file in a specified directory.
-   **High-Quality Conversion**: Converts FLAC to MP3 V0, a high-quality variable bitrate format.
-   **Metadata Preservation**: Retains all album tags and embedded album art during conversion.
-   **Intelligent Naming**: Creates a new folder for each album, automatically replacing "FLAC" or similar keywords with "MP3-v0" for clear organization.
-   **Automatic Cleanup**: Deletes the extracted FLAC files and temporary folders after a successful conversion, leaving only the original `.rar` and the new MP3 folder.
-   **Detailed Progress**: Provides clear progress bars to show the overall progress and the status of each individual conversion.

## Prerequisites

Before running the script, you need to install the necessary command-line tools and Python libraries.

### 1. Install Command-Line Tools

These tools are essential for handling RAR archives and audio conversion. You can install them using the following commands on your Ubuntu server:

```bash
# Update package list
sudo apt update

# Install the unrar tool to extract .rar archives
sudo apt install unrar

# Install ffmpeg for FLAC to MP3 conversion
sudo apt install ffmpeg
```

### 2. Install Python Libraries

The script requires the `tqdm` library for its progress bars. You can install it using `pip` within your Python environment.

```bash
pip install tqdm
```

## How to Use the Script

### Step 1: Save the Script

Save the provided Python code into a file named `music_converter.py`.

```python
import os
import shutil
import subprocess
from pathlib import Path
from tqdm import tqdm

def convert_rar_to_mp3(directory_path):
    """
    Processes all .rar files in a directory, extracts FLACs, converts them to MP3 V0,
    and organizes the output.
    """
    rar_files = [f for f in Path(directory_path).iterdir() if f.is_file() and f.suffix.lower() == '.rar']
    
    if not rar_files:
        print(f"No .rar files found in {directory_path}. Exiting.")
        return

    print(f"Found {len(rar_files)} .rar files to process.")

    for rar_file in tqdm(rar_files, desc="Processing RAR archives"):
        try:
            process_single_rar(rar_file)
        except Exception as e:
            print(f"\n[ERROR] Failed to process '{rar_file.name}': {e}")
            continue

    print("\nAll files processed successfully.")

def process_single_rar(rar_file_path):
    """
    Handles the full workflow for a single .rar file.
    """
    rar_name = rar_file_path.stem
    output_dir_name = rar_name.replace("FLAC", "MP3-v0").replace("flac", "MP3-v0").replace("16", "").strip()
    
    # Create a temporary directory for extraction
    temp_extract_dir = rar_file_path.parent / f"{rar_name}_temp_extract"
    temp_extract_dir.mkdir(exist_ok=True)
    
    # Step 1: Extract the RAR file
    print(f"\n[INFO] Extracting '{rar_file_path.name}'...")
    try:
        subprocess.run(['unrar', 'x', rar_file_path, str(temp_extract_dir) + '/'], 
                       check=True, 
                       capture_output=True,
                       text=True)
    except subprocess.CalledProcessError as e:
        print(f"Extraction failed. Stderr: {e.stderr}")
        raise
    
    # Find all FLAC files in the temporary directory (recursive search)
    flac_files = list(temp_extract_dir.rglob('*.flac'))

    if not flac_files:
        print(f"No FLAC files found in '{rar_name}'. Skipping conversion.")
        shutil.rmtree(temp_extract_dir)
        return

    # Create the final output directory
    final_output_dir = rar_file_path.parent / output_dir_name
    final_output_dir.mkdir(exist_ok=True)

    # Step 2: Convert FLAC files to MP3 V0
    print(f"[INFO] Converting {len(flac_files)} FLAC files to MP3...")
    for flac_file in tqdm(flac_files, desc=f"Converting '{rar_name}'", leave=False):
        mp3_file_name = flac_file.with_suffix('.mp3').name
        mp3_file_path = final_output_dir / mp3_file_name
        
        try:
            # ffmpeg command to convert to MP3 V0 while preserving all metadata and album art
            subprocess.run([
                'ffmpeg',
                '-i', str(flac_file),
                '-codec:a', 'libmp3lame',
                '-q:a', '0',  # V0 encoding
                '-map_metadata', '0', # Keep all metadata
                '-codec:v', 'copy', # Copy album art without re-encoding
                str(mp3_file_path)
            ], check=True, capture_output=True, text=True)
        except subprocess.CalledProcessError as e:
            print(f"Conversion of '{flac_file.name}' failed. Stderr: {e.stderr}")
            raise

    # Step 3: Clean up the extracted files and directory
    print(f"[INFO] Deleting temporary files for '{rar_name}'...")
    shutil.rmtree(temp_extract_dir)
    
    print(f"[SUCCESS] Converted '{rar_file_path.name}' to '{final_output_dir.name}'.")

# --- Usage ---
if __name__ == "__main__":
    # Replace with the path to your directory of .rar files
    music_directory = '/path/to/your/music_collection'
    convert_rar_to_mp3(music_directory)
```

### Step 2: Configure the Directory Path

Open the `music_converter.py` file with a text editor and edit the `music_directory` variable near the bottom of the script.

```python
# Replace with the path to your directory of .rar files
music_directory = '/path/to/your/music_collection'
```

**Important**: Make sure to use the full, absolute path to the folder containing your `.rar` archives. For example: `/home/ubuntu/albums`.

### Step 3: Run the Script

Execute the script from your terminal. Navigate to the directory where you saved `music_converter.py` and run the following command:

```bash
python3 music_converter.py
```

The script will begin processing each `.rar` file one by one, displaying its progress in the terminal.

## How It Works Under the Hood

The script's logic is broken down into these key phases:

1.  **Iterate over Archives**: It finds all `.rar` files in the specified directory.
2.  **Extraction**: It uses the `unrar` command-line tool to extract the contents into a temporary directory.
3.  **Conversion**: For each extracted FLAC file, it invokes `ffmpeg` with specific parameters:
    * `-i`: Specifies the input FLAC file.
    * `-codec:a libmp3lame`: Selects the LAME encoder for MP3.
    * `-q:a 0`: Sets the quality to V0, which provides the best possible quality for variable bitrate MP3.
    * `-map_metadata 0`: Copies all metadata from the source file.
    * `-codec:v copy`: Copies the album art without re-encoding, preserving its quality and saving time.
4.  **Cleanup**: Once all FLAC files from an archive have been successfully converted, the temporary extraction directory and its contents are automatically deleted.
5.  **Output**: The resulting MP3 files, along with all their tags and album art, are placed in a newly created folder with a clean, descriptive name.

This process ensures that your music library is converted efficiently and accurately, leaving you with a well-organized and high-quality MP3 collection.
"""

print(README_CONTENT)
