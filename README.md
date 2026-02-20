# Python-Overlay

from pynq import Overlay
from pynq import MMIO
import numpy as np
import time
import os
print("---------------------------------------------------------")
print("STEP 1: SYSTEM CHECK & HARDWARE LOADING")
print("---------------------------------------------------------")

# Verify files exist before loading
if not os.path.exists("cnn_accel.bit"):
    print("❌ ERROR: 'cnn_accel.bit' not found in current folder!")
    raise FileNotFoundError("Upload .bit file")
if not os.path.exists("cnn_accel.hwh"):
    print("❌ ERROR: 'cnn_accel.hwh' not found in current folder!")
    raise FileNotFoundError("Upload .hwh file")

print("--> Loading Bitstream 'cnn_accel.bit'...")
try:
    overlay = Overlay("cnn_accel.bit") 
    print("✅ Bitstream downloaded to FPGA successfully.")
except Exception as e:
    print(f"❌ CRITICAL ERROR during Overlay load: {e}")
    print("   (This usually means Power Failure or Wrong Zynq Board Settings)")
    raise

print("--> Connecting to AXI IP at 0x43C00000...")
try:
    fpga = MMIO(0x43C00000, 0x10000)
    # Try reading a register to verify connection (Control Reg should be 0 initially)
    test_read = fpga.read(0x28) 
    print(f"✅ Connection verified. Control Register value: {test_read}")
except Exception as e:
    print(f"❌ ERROR connecting to MMIO: {e}")
    raise

# REGISTER MAP
REG_W = [0x00, 0x04, 0x08, 0x0C, 0x10, 0x14, 0x18, 0x1C, 0x20]
REG_PIXEL = 0x24 
REG_CTRL = 0x28 
REG_RESULT = 0x2C 
REG_VALID = 0x30 
print("\n---------------------------------------------------------")
print("STEP 2: DATA GENERATION (NO FILE NEEDED)")
print("---------------------------------------------------------")

# Instead of loading a file, we create a "Fake" image in memory.
# We create 10 images of size 28x28 with random pixel values (0-255).
print("--> Generating synthetic test data in memory...", flush=True)
try:
    # Random noise image
    raw_images = np.random.randint(0, 255, (10, 28, 28), dtype=np.uint8)
    print(f"✅ Generated 10 synthetic images. Shape: {raw_images.shape}", flush=True)
except Exception as e:
    print(f"❌ RAM Error generating data: {e}")
    raise

print("--> Pre-processing (Padding 28x28 -> 32x32)...", flush=True)
print("\n---------------------------------------------------------")
print("STEP 3: CONFIGURING ACCELERATOR")
print("---------------------------------------------------------")

weights = [2, -1, 0, -1, 4, -1, 0, -1, 2]
print(f"--> Sending Weights: {weights}")

try:
    for i in range(9):
        fpga.write(REG_W[i], int(weights[i]))
    # Read back one weight to confirm
    read_back = fpga.read(REG_W[4])
    print(f"✅ Weights Written. Verification Read (Center Weight): {read_back} (Expected 4)")
    if read_back != 4:
        print("⚠️ WARNING: Readback failed! FPGA might be frozen.")
except Exception as e:
    print(f"❌ ERROR writing weights: {e}")
    print("   (This suggests the AXI Clock is dead. Check Block Diagram)")
    raise
    print("\n---------------------------------------------------------")
print("STEP 4: EXECUTING CONVOLUTION")
print("---------------------------------------------------------")

def run_fpga(image):
    # Convert image to Q1.7 Integers (0-255)
    pixels = (image.flatten() * 128).astype(int)
    pixels = np.clip(pixels, 0, 255)
    
    results = []
    
    # Enable Accelerator
    fpga.write(REG_CTRL, 1)
    
    # Stream Pixels
    for p in pixels:
        fpga.write(REG_PIXEL, int(p))
        if fpga.read(REG_VALID) == 1:
            val = fpga.read(REG_RESULT)
            if val > 32767: val -= 65536
            results.append(val)
            
    fpga.write(REG_CTRL, 0)
    return results

print("--> Processing Image #0 on FPGA...")
start = time.time()
try:
    hw_output = run_fpga(test_images_norm[0])
    end = time.time()
    print(f"✅ FPGA Finished in {end - start:.4f} seconds.")
    print(f"--> Output Feature Map Size: {len(hw_output)}")
except Exception as e:
    print(f"❌ ERROR during execution: {e}")
    raise
print("\n---------------------------------------------------------")
print("STEP 5: SOFTWARE VERIFICATION")
print("---------------------------------------------------------")

def run_cpu_verification(image, kernel):
    print("--> Calculating CPU Reference (Golden Check)...")
    img_int = (image.flatten() * 128).astype(int)
    img_int = np.clip(img_int, 0, 255).reshape(32,32)
    k_matrix = np.array(kernel).reshape(3,3)
    
    output = []
    rows, cols = 30, 30
    
    for r in range(rows):
        for c in range(cols):
            patch = img_int[r:r+3, c:c+3]
            res = np.sum(patch * k_matrix)
            output.append(res)
    return output

sw_output = run_cpu_verification(test_images_norm[0], weights)
print("✅ CPU Calculation Complete.")
print("\n====================================")
print(" FINAL RESULTS REPORT")
print("====================================")

if len(hw_output) == 0:
    print("❌ FAILURE: FPGA returned 0 results.")
    print("   Check: Is REG_CTRL being set? Is the Valid signal wired correctly?")
else:
    hw_max = max(hw_output)
    sw_max = max(sw_output)

    print(f"FPGA Max Value: {hw_max}")
    print(f"CPU  Max Value: {sw_max}")

    if hw_max == sw_max:
        print("✅ SUCCESS: Hardware matches Software perfectly!")
    else:
        print(f"❌ MISMATCH: Difference is {abs(hw_max - sw_max)}")
        print("   Hint: Check if you are mixing Signed/Unsigned logic in Verilog.")
