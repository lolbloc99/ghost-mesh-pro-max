
import streamlit as st
import cv2
import numpy as np
import tempfile
import os
import uuid
import subprocess
from pydub import AudioSegment

def generate_dynamic_mesh(w, h, spacing=40, thickness=1, offset_x=0, offset_y=0, contrast=1.0):
    mesh = np.zeros((h, w, 3), dtype=np.uint8)
    base_color = np.random.randint(180, 255)
    color = tuple([min(255, int(base_color * contrast)) for _ in range(3)])
    for x in range(offset_x % spacing, w, spacing):
        cv2.line(mesh, (x, 0), (x, h), color, thickness)
    for y in range(offset_y % spacing, h, spacing):
        cv2.line(mesh, (0, y), (w, y), color, thickness)
    return mesh

def combine_meshes(meshes):
    combined = meshes[0]
    for m in meshes[1:]:
        combined = cv2.addWeighted(combined, 0.5, m, 0.5, 0)
    return combined

def add_dynamic_glow(frame):
    blur = cv2.GaussianBlur(frame, (9, 9), 0)
    return cv2.addWeighted(frame, 0.85, blur, 0.25, 0)

def add_noise(frame, intensity=0.03):
    noise = np.random.normal(0, intensity * 255, frame.shape).astype(np.int16)
    noisy = np.clip(frame.astype(np.int16) + noise, 0, 255).astype(np.uint8)
    return noisy

def vary_brightness(frame, factor=0.98):
    return np.clip(frame * factor, 0, 255).astype(np.uint8)

def apply_warp(frame, strength=5):
    h, w = frame.shape[:2]
    map_y, map_x = np.indices((h, w), dtype=np.float32)
    dx = np.sin(np.linspace(0, np.pi * 2, w)) * strength
    dy = np.cos(np.linspace(0, np.pi * 2, h)) * strength
    map_x = map_x + dx[np.newaxis, :].astype(np.float32)
    map_y = map_y + dy[:, np.newaxis].astype(np.float32)
    return cv2.remap(frame, map_x, map_y, interpolation=cv2.INTER_LINEAR)

def audio_furtive_effect(audio_path):
    sound = AudioSegment.from_file(audio_path)
    sound = sound + np.random.randint(-1, 1)  # micro variation
    temp_path = audio_path.replace(".aac", "_furtif.aac")
    sound.export(temp_path, format="adts")
    return temp_path

def process_single_video(input_path, output_path, alpha, warp_strength):
    cap = cv2.VideoCapture(input_path)
    fps = cap.get(cv2.CAP_PROP_FPS)
    width = int(cap.get(cv2.CAP_PROP_FRAME_WIDTH))
    height = int(cap.get(cv2.CAP_PROP_FRAME_HEIGHT))
    count = int(cap.get(cv2.CAP_PROP_FRAME_COUNT))
    fourcc = cv2.VideoWriter_fourcc(*'mp4v')
    out = cv2.VideoWriter(output_path, fourcc, fps, (width, height))

    for i in range(count):
        ret, frame = cap.read()
        if not ret:
            break
        meshes = []
        for j in range(4):
            spacing = np.random.choice([30, 40, 50])
            offset_x = (i * (j + 1) * 2) % spacing
            offset_y = (i * (j + 1) * 3) % spacing
            contrast = np.random.uniform(0.8, 1.2)
            mesh = generate_dynamic_mesh(width, height, spacing=spacing,
                                         offset_x=offset_x, offset_y=offset_y,
                                         contrast=contrast)
            meshes.append(mesh)
        combined_mesh = combine_meshes(meshes)
        mesh_applied = cv2.addWeighted(frame, 1.0, combined_mesh, alpha, 0)
        mod = apply_warp(mesh_applied, warp_strength)
        mod = add_dynamic_glow(mod)
        mod = add_noise(mod)
        mod = vary_brightness(mod, factor=np.random.uniform(0.95, 1.05))
        out.write(mod)
    cap.release()
    out.release()

    audio_path = input_path.replace(".mp4", ".aac")
    subprocess.run(["ffmpeg", "-y", "-i", input_path, "-vn", "-acodec", "copy", audio_path])

    mod_audio = audio_furtive_effect(audio_path)
    final_path = output_path.replace(".mp4", "_final.mp4")
    subprocess.run([
        "ffmpeg", "-y", "-i", output_path, "-i", mod_audio,
        "-c:v", "copy", "-c:a", "aac", "-shortest", final_path
    ])
    return final_path

st.title("🎞️ Traitement batch avec audio furtif")

alpha = st.slider("🎚️ Intensité du mesh", 0.0, 0.5, 0.1, 0.01)
warp = st.slider("💫 Intensité du warp", 0, 20, 5, 1)

uploaded_files = st.file_uploader("Uploader plusieurs vidéos", type=["mp4"], accept_multiple_files=True)

if uploaded_files:
    download_links = []
    for uploaded_file in uploaded_files:
        with tempfile.NamedTemporaryFile(delete=False, suffix=".mp4") as tmp_input:
            tmp_input.write(uploaded_file.read())
            tmp_input_path = tmp_input.name
        output_path = os.path.join(tempfile.gettempdir(), f"mod_{uuid.uuid4().hex}.mp4")
        with st.spinner(f"Traitement en cours : {uploaded_file.name}"):
            result_path = process_single_video(tmp_input_path, output_path, alpha, warp)
            download_links.append(result_path)

    for path in download_links:
        with open(path, "rb") as file:
            st.download_button("📥 Télécharger", data=file, file_name=os.path.basename(path), mime="video/mp4")
