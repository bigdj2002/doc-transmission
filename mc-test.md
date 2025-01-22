
def calculate_vmaf(ffmpeg_path, original_video, test_video, vmaf_model, log_path=None):
    """
    Calculate VMAF score between an original video and a test video using FFmpeg.

    :param ffmpeg_path: Path to the FFmpeg executable.
    :param original_video: Path to the original (reference) video file.
    :param test_video: Path to the test (distorted) video file.
    :param vmaf_model: Path to the VMAF model file (e.g., "vmaf_v0.6.1.json").
    :param log_path: Path to save the VMAF log (optional).
    :return: VMAF score as a float.
    """
    # Build FFmpeg command
    cmd = [
        ffmpeg_path,
        "-i", test_video,
        "-i", original_video,
        "-lavfi", f"[0:v:0][1:v:0]libvmaf=model_path={vmaf_model}:log_fmt=json",
        "-f", "null", "-"
    ]

    # Add log file output if specified
    if log_path:
        cmd[-4] = f"[0:v:0][1:v:0]libvmaf=model_path={vmaf_model}:log_fmt=json:log_path={log_path}"

    try:
        # Execute the FFmpeg command
        result = subprocess.run(cmd, stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True)
        # Extract VMAF score from the FFmpeg output
        for line in result.stderr.splitlines():
            if "VMAF score:" in line:
                return float(line.split(":")[-1].strip())
    except Exception as e:
        print(f"Error calculating VMAF: {e}")
    return None
