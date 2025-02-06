import os
import hashlib
from tqdm import tqdm  # For progress bar

# Paths to the pendrives (update if needed)
pendrives = ["/Volumes/OyeBusy-MacData-256G",
             "/Volumes/sandisk32", "/Volumes/Akash256"]

# Folders to ignore
ignore_folders = {"node_modules", "venv", "__pycache__", ".git"}

# Dictionary to store file hashes and paths
file_records = {}


def get_file_hash(file_path):
    """Generate SHA256 hash for a file"""
    hasher = hashlib.sha256()
    try:
        with open(file_path, "rb") as f:
            while chunk := f.read(8192):
                hasher.update(chunk)
        return hasher.hexdigest()
    except Exception as e:
        print(f"⚠️ Error hashing {file_path}: {e}")
        return None


def scan_pendrive(base_path):
    """Scan all files in the pendrive, skipping ignored folders"""
    files = []
    for root, dirs, filenames in os.walk(base_path):
        # Exclude ignored directories
        dirs[:] = [d for d in dirs if d not in ignore_folders]
        for filename in filenames:
            file_path = os.path.join(root, filename)
            files.append(file_path)
    return files


def get_disk_usage(path):
    """Get available disk space"""
    stat = os.statvfs(path)
    return stat.f_frsize * stat.f_bavail  # Available free space in bytes


def find_and_delete_duplicates():
    """Find and delete duplicate files"""
    initial_disk_space = {p: get_disk_usage(p) for p in pendrives}

    for pendrive in pendrives:
        if not os.path.exists(pendrive):
            print(f"❌ Skipping: {pendrive} (not found)")
            continue
        print(f"🔍 Scanning {pendrive}...")

        files = scan_pendrive(pendrive)
        
        print(f"📂 Found {len(files)} files in {pendrive}. Processing...")

        # Process files with a progress bar
        for file_path in tqdm(files, desc=f"Processing {pendrive}", unit="file"):
            try:
                file_size = os.path.getsize(file_path)
                file_name = os.path.basename(file_path)
                file_hash = get_file_hash(file_path)

                if not file_hash:
                    continue

                # Create a unique key for duplicate detection
                key = (file_name, file_size, file_hash)

                if key in file_records:
                    print(f"❌ Duplicate found: {file_path} (Deleting...)")
                    try:
                        os.remove(file_path)
                        if os.path.exists(file_path):
                            print(f"⚠️ Failed to delete: {file_path}")
                        else:
                            print(f"✅ Successfully deleted: {file_path}")
                    except Exception as e:
                        print(f"⚠️ Error deleting {file_path}: {e}")
                else:
                    file_records[key] = file_path  # Store the first occurrence

            except Exception as e:
                print(f"⚠️ Error processing {file_path}: {e}")

    final_disk_space = {p: get_disk_usage(p) for p in pendrives}

    for p in pendrives:
        print(f"📊 Disk space before: {initial_disk_space[p] / (1024**3):.2f} GB")
        print(f"📊 Disk space after: {final_disk_space[p] / (1024**3):.2f} GB")


# Run the duplicate finder
find_and_delete_duplicates()
print("✅ Duplicate file cleanup complete!")
