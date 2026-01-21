"""
Smart Campus Navigation Assistant
Merged & improved version:
 - Graph-based campus model with distances & directions
 - Shortest-path (Dijkstra) by distance
 - Saves a map image highlighting the route
 - Generates step-by-step TTS audio (gTTS)
 - QR code linking to a Drive map
 - Outputs stored in ./navigation_output/
Works in Jupyter and CLI. Requires:
pip install networkx matplotlib gTTS qrcode[pil] pillow
"""

import os
import sys
import time
import shutil
import networkx as nx
import matplotlib.pyplot as plt

# Jupyter display imports guarded (won't error on CLI)
try:
    from IPython.display import display, Audio, Image, Markdown
    IPYTHON_AVAILABLE = True
except Exception:
    IPYTHON_AVAILABLE = False

# ----------------- CONFIG & DATA -----------------
OUTPUT_DIR = "navigation_output"
AUDIO_DIR = os.path.join(OUTPUT_DIR, "audio")
IMG_PATH = os.path.join(OUTPUT_DIR, "college_map.png")
QR_PATH = os.path.join(OUTPUT_DIR, "route_qr.png")
COMBINED_AUDIO = os.path.join(OUTPUT_DIR, "route_instructions.mp3")

# Drive link (use your provided drive link)
DRIVE_LINK = "https://drive.google.com/file/d/1lNnQzOFhpmxsceMFNiN4bJoMqNuV8Q7n/view?usp=drivesdk"

# Room numbers (merged / extended)
room_numbers = {
    "Entrance": None,
    "Office": "209A",
    "Principal Office": "212",
    "Seminar Hall 2": "208",
    "CC1": "201",
    "CC2": "201",
    "CC3": "205",
    "Seminar Hall 1": "307",
    "CSE Dept": "316",
    "ECE Dept": "318",
    "EEE Dept": "417",
    # Additional example nodes from Evaluation.docx:
    "Library": "301",
    "Auditorium": "201",
    "Canteen": "101",
    "Hostel": "H1"
}

# Floor mapping (based on first digit of room number when available)
floor_mapping = {
    "1": "1st Floor",
    "2": "2nd Floor",
    "3": "3rd Floor",
    "4": "4th Floor",
    "5": "5th Floor"
}

# Graph edges with distance and direction (combined from your versions)
edges_with_attrs = [
    ("Entrance", "Office", 10, "straight"),
    ("Office", "Principal Office", 5, "right"),
    ("Office", "Seminar Hall 2", 8, "left"),
    ("Office", "CC1", 12, "straight"),
    ("CC1", "CC2", 6, "right"),
    ("CC2", "CC3", 7, "straight"),
    ("CC3", "Seminar Hall 1", 10, "left"),
    ("Seminar Hall 1", "CSE Dept", 15, "right"),
    ("CSE Dept", "ECE Dept", 10, "left"),
    ("ECE Dept", "EEE Dept", 20, "straight"),
    # Additional edges from Evaluation.docx dataset
    ("Entrance", "CSE Dept", 50, "straight"),
    ("CSE Dept", "Library", 100, "right"),
    ("Library", "Hostel", 150, "left"),
    ("CSE Dept", "ECE Dept", 60, "left"),   # duplicate with different weight - we'll keep minimal
    ("ECE Dept", "Auditorium", 90, "right"),
    ("Library", "Canteen", 80, "right"),
    ("Canteen", "Hostel", 70, "left")
]

# Positions used for visual layout (tweak as desired)
POS = {
    "Entrance": (0, 0),
    "Office": (0, 1),
    "Principal Office": (1, 1),
    "Seminar Hall 2": (-1, 1),
    "CC1": (0, 2),
    "CC2": (1, 2),
    "CC3": (2, 2),
    "Seminar Hall 1": (0, 3),
    "CSE Dept": (1, 3),
    "ECE Dept": (2, 3),
    "EEE Dept": (1, 4),
    "Library": (-1, 3),
    "Auditorium": (2, 4),
    "Canteen": (-1, 2),
    "Hostel": (-2, 3)
}

# ----------------- SETUP OUTPUT FOLDERS -----------------
def prepare_output_dirs():
    if os.path.exists(OUTPUT_DIR):
        # keep existing but clear audio directory
        if os.path.exists(AUDIO_DIR):
            shutil.rmtree(AUDIO_DIR)
    os.makedirs(AUDIO_DIR, exist_ok=True)

# ----------------- BUILD GRAPH -----------------
def build_graph():
    G = nx.DiGraph()
    # Add edges (if multiple edges exist, keep the smallest distance)
    for u, v, dist, direction in edges_with_attrs:
        if G.has_edge(u, v):
            current = G[u][v]["distance"]
            if dist < current:
                G[u][v]["distance"] = dist
                G[u][v]["direction"] = direction
        else:
            G.add_edge(u, v, distance=dist, direction=direction)
    # make sure nodes exist even if no edges added
    for node in room_numbers.keys():
        if node not in G:
            G.add_node(node)
    return G

# ----------------- ROUTE / INSTRUCTIONS -----------------
def build_instruction_steps(G, path):
    steps = []
    total_distance = 0
    for i in range(len(path) - 1):
        u, v = path[i], path[i+1]
        edge = G[u][v]
        dist = edge.get("distance", None)
        direction = edge.get("direction", "forward")
        if dist is not None:
            steps.append(f"From {u}, go {direction} for {dist} meters to reach {v}.")
            total_distance += float(dist)
        else:
            steps.append(f"From {u}, go {direction} to {v}.")
    final = path[-1]
    room = room_numbers.get(final)
    if room:
        steps.append(f"You have arrived at {final} (Room: {room}).")
    else:
        steps.append(f"You have arrived at {final}.")
    steps.append(f"Total estimated distance: {int(total_distance)} meters.")
    return steps, total_distance

# ----------------- TTS -----------------
def save_tts_steps(steps):
    filenames = []
    for i, text in enumerate(steps, start=1):
        safe_name = os.path.join(AUDIO_DIR, f"step_{i}.mp3")
        tts = gTTS(text)
        tts.save(safe_name)
        filenames.append(safe_name)
    # Optionally concatenate into combined file (simple file concat works for mp3 if same encoding)
    # We'll create a combined file by binary append (works in many cases, but not guaranteed for all encoders).
    with open(COMBINED_AUDIO, "wb") as wfd:
        for f in filenames:
            with open(f, "rb") as fd:
                wfd.write(fd.read())
    return filenames, COMBINED_AUDIO

# ----------------- QR CODE -----------------
def generate_qr(link, filename=QR_PATH):
    qr = qrcode.QRCode(
        version=6,
        error_correction=qrcode.constants.ERROR_CORRECT_H,
        box_size=8,
        border=4
    )
    qr.add_data(link)
    qr.make(fit=True)
    img = qr.make_image(fill_color="black", back_color="white")
    img.save(filename)
    return filename

# ----------------- MAP IMAGE WITH HIGHLIGHTED ROUTE -----------------
def draw_map_and_highlight(G, path=None, save_path=IMG_PATH):
    plt.figure(figsize=(9, 7))
    # labels show name and room if available
    labels = {}
    for node in G.nodes():
        rn = room_numbers.get(node)
        labels[node] = f"{node}\n({rn})" if rn else node

    # base nodes and edges
    nx.draw_networkx_nodes(G, POS, node_color="lightblue", node_size=1800)
    nx.draw_networkx_labels(G, POS, labels=labels, font_size=9, font_weight='bold')

    # Draw all edges lighter
    all_edges = list(G.edges())
    nx.draw_networkx_edges(G, POS, edgelist=all_edges, arrows=True, alpha=0.3)

    # If route provided, overlay it
    if path and len(path) > 1:
        route_edges = [(path[i], path[i+1]) for i in range(len(path)-1)]
        nx.draw_networkx_edges(G, POS, edgelist=route_edges, arrows=True, width=3.0, edge_color='red')
        # highlight nodes in route
        nx.draw_networkx_nodes(G, POS, nodelist=path, node_color='orange', node_size=2000)
    plt.title("Smart Campus Navigation Map (highlighted route)", fontsize=14, fontweight='bold')
    plt.axis('off')
    plt.tight_layout()
    plt.savefig(save_path, dpi=150)
    plt.close()
    return save_path

# ----------------- MAIN NAVIGATION FUNCTION -----------------
def navigate(start, end, voice=True, show_outputs_in_notebook=True):
    prepare_output_dirs()
    G = build_graph()

    # Basic validity
    if start not in G or end not in G:
        raise ValueError(f"Start or end not found in campus graph. Start: {start}, End: {end}")

    try:
        path = nx.shortest_path(G, source=start, target=end, weight="distance")
    except nx.NetworkXNoPath:
        if IPYTHON_AVAILABLE and show_outputs_in_notebook:
            display(Markdown("**❌ No path found between the given locations.**"))
        else:
            print("❌ No path found between the given locations.")
        return None

    # Build human-friendly steps
    steps, total_dist = build_instruction_steps(G, path)

    # Display textual steps
    if IPYTHON_AVAILABLE and show_outputs_in_notebook:
        display(Markdown("### 📍 Route Directions"))
        display(Markdown("\n\n".join(steps)))
    else:
        print("\nRoute Directions:")
        for s in steps:
            print(s)

    # Save TTS
    audio_files = []
    if voice:
        try:
            audio_files, combined = save_tts_steps(steps)
            if IPYTHON_AVAILABLE and show_outputs_in_notebook:
                display(Markdown("### 🔊 Playing instructions (combined)"))
                display(Audio(combined, autoplay=True))
            else:
                print(f"✅ Audio instructions saved to {combined}")
        except Exception as e:
            print("⚠️ TTS generation failed:", e)

    # Draw map with highlighted path
    img_file = draw_map_and_highlight(G, path=path)
    if IPYTHON_AVAILABLE and show_outputs_in_notebook:
        display(Image(img_file))
    else:
        print(f"✅ Map image saved to {img_file}")

    # Generate QR code linking to Drive (or a URL you supply)
    qr_file = generate_qr(DRIVE_LINK)
    if IPYTHON_AVAILABLE and show_outputs_in_notebook:
        display(Markdown("✅ Scan the QR code to open the full route map on Google Drive."))
        display(Image(qr_file))
    else:
        print(f"✅ QR code saved to {qr_file}")

    # Summary info
    summary = {
        "path": path,
        "steps": steps,
        "total_distance": total_dist,
        "map_image": img_file,
        "qr_code": qr_file,
        "audio_files": audio_files,
        "combined_audio": COMBINED_AUDIO if voice else None
    }
    return summary

# ----------------- CLI / JUPYTER ENTRYPOINT -----------------
def list_available_destinations():
    return sorted([n for n in room_numbers.keys() if n is not None])

def run_interactive():
    print("Available destinations:")
    for name in list_available_destinations():
        print(" -", name)
    start = "Entrance"
    # Ask user for destination (case-insensitive matching)
    dest_input = input("Enter destination (name): ").strip()
    matches = [name for name in room_numbers if name.lower() == dest_input.lower()]

    if not matches:
        print("❌ No valid destination provided. Try one of the available names exactly as shown.")
        return

    dest = matches[0]
    # Ask for voice or text mode
    mode = input("Mode? (voice/text): ").strip().lower()
    voice = True if mode != "text" else False

    result = navigate(start, dest, voice=voice, show_outputs_in_notebook=IPYTHON_AVAILABLE)
    if result:
        print("\n✅ Navigation finished. Outputs saved to the 'navigation_output' folder.")
        print(" - Map:", result['map_image'])
        print(" - QR code:", result['qr_code'])
        if voice:
            print(" - Combined audio:", result['combined_audio'])

# If run as script, start interactive mode
if __name__ == "__main__":
    run_interactive()
