import subprocess
import os

def calculate_and_verify_bitrate(file_path):
    """
    Calculate the video bitrate using file size and duration, and compare it with Mediainfo's reported bitrate.

    :param file_path: Path to the video file.
    :return: A dictionary containing calculated and reported bitrates, and verification result.
    """
    if not os.path.isfile(file_path):
        raise FileNotFoundError(f"File not found: {file_path}")
    
    try:
        # Run mediainfo command and capture output
        result = subprocess.run(
            ["mediainfo", "--Output=JSON", file_path],
            stdout=subprocess.PIPE,
            text=True
        )
        
        # Parse the JSON output
        import json
        info = json.loads(result.stdout)
        
        # Extract required fields
        general_info = info['media']['track'][0]
        file_size = int(general_info.get("FileSize", 0))  # File size in bytes
        duration = float(general_info.get("Duration", 0)) / 1000  # Duration in seconds
        
        video_info = next(
            (track for track in info['media']['track'] if track["@type"] == "Video"), None
        )
        reported_bitrate = float(video_info.get("BitRate", 0)) / 1000  # Bitrate in kbps
        
        # Calculate bitrate
        if file_size > 0 and duration > 0:
            calculated_bitrate = (file_size * 8) / (duration * 1000)  # kbps
        else:
            raise ValueError("Invalid file size or duration for bitrate calculation.")
        
        # Verification
        is_match = abs(calculated_bitrate - reported_bitrate) < 0.1 * reported_bitrate  # 10% tolerance
        
        # Return results
        return {
            "calculated_bitrate_kbps": round(calculated_bitrate, 2),
            "reported_bitrate_kbps": round(reported_bitrate, 2),
            "verification_passed": is_match
        }
    
    except Exception as e:
        raise RuntimeError(f"Error while processing the file: {e}")

# Example usage
file_path = "/path/to/your/video.mp4"
try:
    result = calculate_and_verify_bitrate(file_path)
    print(f"Calculated Bitrate: {result['calculated_bitrate_kbps']} kbps")
    print(f"Reported Bitrate: {result['reported_bitrate_kbps']} kbps")
    print(f"Verification Passed: {result['verification_passed']}")
except Exception as e:
    print(e)
