# readme_generator.py
# This script prints the full Markdown content for the README.md file.

README_CONTENT = """# Interactive RAR to MP3 Converter for Music Albums

This is an interactive script to process a directory of music albums archived in `.rar` format. It provides a menu-driven interface to configure settings before extracting FLAC files, converting them to MP3s, and organizing the output into a clean directory structure.

The script is designed for Linux environments, specifically Ubuntu Server 24.04, and uses standard command-line tools for maximum efficiency and reliability.

## Features

-   **Interactive Menu**: Allows you to set the input directory, output directory, and MP3 encoding quality before starting the conversion.
-   **High-Quality Conversion**: Converts FLAC to MP3 with user-selectable quality presets (V0, V2, or 320kbps CBR).
-   **Metadata Preservation**: Retains all album tags and embedded album art during conversion.
-   **Intelligent Naming**: Creates a new folder for each album, automatically replacing "FLAC" or similar keywords with "MP3-v0" or a similar descriptive tag for clear organization.
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

def convert_rar_to_mp3(input_directory, output_directory, encoding_preset):
    """
    Processes all .rar files in a directory, extracts FLACs, converts them to MP3,
    and organizes the output based on user settings.
    """
    input_path = Path(input_directory)
    output_path = Path(output_directory)

    # Ensure output directory exists
    output_path.mkdir(exist_ok=True, parents=True)

    rar_files = [f for f in input_path.iterdir() if f.is_file() and f.suffix.lower() == '.rar']
    
    if not rar_files:
        print(f"No .rar files found in {input_path}. Exiting.")
        return

    print(f"Found {len(rar_files)} .rar files to process.")
    print(f"Using output directory: {output_path}")
    print(f"Using encoding preset: {encoding_preset['name']} ({encoding_preset['quality_flag']})")

    for rar_file in tqdm(rar_files, desc="Processing RAR archives"):
        try:
            process_single_rar(rar_file, output_path, encoding_preset)
        except Exception as e:
            print(f"\n[ERROR] Failed to process '{rar_file.name}': {e}")
            continue

    print("\nAll files processed successfully.")

def process_single_rar(rar_file_path, output_path, encoding_preset):
    """
    Handles the full workflow for a single .rar file.
    """
    rar_name = rar_file_path.stem
    output_dir_name = rar_name.replace("FLAC", encoding_preset['name']).replace("flac", encoding_preset['name']).replace("16", "").strip()
    
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
    final_output_dir = output_path / output_dir_name
    final_output_dir.mkdir(exist_ok=True)

    # Step 2: Convert FLAC files to MP3 with chosen preset
    print(f"[INFO] Converting {len(flac_files)} FLAC files to MP3...")
    for flac_file in tqdm(flac_files, desc=f"Converting '{rar_name}'", leave=False):
        mp3_file_name = flac_file.with_suffix('.mp3').name
        mp3_file_path = final_output_dir / mp3_file_name
        
        try:
            ffmpeg_command = [
                'ffmpeg',
                '-i', str(flac_file),
                '-codec:a', 'libmp3lame',
                '-map_metadata', '0', # Keep all metadata
                '-codec:v', 'copy', # Copy album art without re-encoding
                str(mp3_file_path)
            ]
            
            # Add the quality flag based on the chosen preset
            ffmpeg_command.insert(5, encoding_preset['quality_flag'])
            ffmpeg_command.insert(6, encoding_preset['quality_value'])

            subprocess.run(ffmpeg_command, check=True, capture_output=True, text=True)
            
        except subprocess.CalledProcessError as e:
            print(f"Conversion of '{flac_file.name}' failed. Stderr: {e.stderr}")
            raise

    # Step 3: Clean up the extracted files and directory
    print(f"[INFO] Deleting temporary files for '{rar_name}'...")
    shutil.rmtree(temp_extract_dir)
    
    print(f"[SUCCESS] Converted '{rar_file_path.name}' to '{final_output_dir.name}'.")

def get_encoding_preset():
    """
    Prompts the user to select an MP3 encoding preset.
    """
    presets = {
        '1': {'name': 'MP3-v0', 'quality_flag': '-q:a', 'quality_value': '0'},
        '2': {'name': 'MP3-v2', 'quality_flag': '-q:a', 'quality_value': '2'},
        '3': {'name': 'MP3-320kbps-CBR', 'quality_flag': '-b:a', 'quality_value': '320k'}
    }
    
    while True:
        print("\\n--- MP3 Encoding Settings ---")
        print("1. V0 (Variable Bitrate - Highest Quality)")
        print("2. V2 (Variable Bitrate - High Quality, Smaller Size)")
        print("3. 320kbps CBR (Constant Bitrate - Maximum Quality, Larger Size)")
        choice = input("Enter your choice (1, 2, or 3): ")
        if choice in presets:
            return presets[choice]
        else:
            print("Invalid choice. Please enter a number between 1 and 3.")

def main():
    """
    Main interactive loop for the script.
    """
    input_directory = './'
    output_directory = './'
    encoding_preset = {'name': 'MP3-v0', 'quality_flag': '-q:a', 'quality_value': '0'} # Default to V0

    while True:
        print("\\n--- Main Menu ---")
        print(f"1. Set Input Directory (Current: {input_directory})")
        print(f"2. Set Output Directory (Current: {output_directory})")
        print(f"3. Configure MP3 Encoding (Current: {encoding_preset['name']})")
        print("4. Start Conversion")
        print("5. Exit")
        
        choice = input("Enter your choice: ")
        
        if choice == '1':
            new_dir = input("Enter the path to the input directory: ")
            if Path(new_dir).is_dir():
                input_directory = new_dir
            else:
                print("Invalid directory path. Please try again.")
        elif choice == '2':
            new_dir = input("Enter the path to the output directory: ")
            output_directory = new_dir # We will create this directory if it doesn't exist
        elif choice == '3':
            encoding_preset = get_encoding_preset()
        elif choice == '4':
            convert_rar_to_mp3(input_directory, output_directory, encoding_preset)
        elif choice == '5':
            print("Exiting script.")
            break
        else:
            print("Invalid choice. Please enter a number between 1 and 5.")

if __name__ == "__main__":
    main()
```

### Step 2: Configure the Script

This script is now interactive and will prompt you for input. You no longer need to edit the file to set the directories; you can do so from the menu.

### Step 3: Run the Script

Execute the script from your terminal. Navigate to the directory where you saved `music_converter.py` and run the following command:

```bash
python3 music_converter.py
```

The script will present you with a main menu to configure your settings before starting the conversion.

## How It Works Under the Hood

The script's logic has been enhanced with a main menu loop (`main()` function) and a helper function to set encoding presets (`get_encoding_preset()`). The core conversion logic is now encapsulated in the `convert_rar_to_mp3` function, which accepts the user-defined `input_directory`, `output_directory`, and `encoding_preset` as arguments. The `ffmpeg` command is dynamically constructed based on your choice of encoding quality.
"""

print(README_CONTENT)
