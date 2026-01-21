# Smart-Campus-Assistant
This project presents a graph-based indoor navigation system designed to assist users in navigating large academic campuses. The campus layout is modeled as a directed graph where locations are represented as nodes and walkable paths as edges. Dijkstra’s algorithm is used to compute the shortest path between a source and destination.  
# Smart Campus Navigation Assistant

A graph-based indoor navigation system designed to help users navigate large academic campuses efficiently without relying on GPS.

## 📌 Overview
Navigating large campuses can be challenging for new students, visitors, and visually impaired users. This project provides an indoor navigation solution using graph-based modeling and shortest-path algorithms. The system computes optimal routes and delivers guidance through visual maps, text instructions, and voice output.

## ⚙️ Key Features
- Graph-based campus representation
- Shortest path computation using Dijkstra’s algorithm
- Visual route plotting with NetworkX and Matplotlib
- Voice-guided navigation using Google Text-to-Speech (gTTS)
- QR code generation for quick mobile access
- GPS-independent and low-cost solution

## 🧠 How It Works
- Campus locations are modeled as nodes and walkable paths as edges
- If no physical distance is provided, all edges are treated as equal weight
- Dijkstra’s algorithm computes the shortest path based on minimum hops
- Navigation instructions are generated in text, voice, and visual form

## 🛠️ Technologies Used
- Python
- NetworkX
- Matplotlib
- Google Text-to-Speech (gTTS)
- QR Code library

## 📊 Applications
- Educational campuses
- Hospitals
- Shopping malls
- Museums
- Large office buildings

## 🚀 Future Enhancements
- Real-time indoor localization
- Dynamic rerouting
- Mobile application support
- Accessibility-based routing
- Augmented reality navigation

## 👨‍💻 Author
K Abhinav Krishna
