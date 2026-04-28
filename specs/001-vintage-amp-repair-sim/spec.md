This is an excellent way to bridge your CS background with hardware. The github/spec-kit is designed for clear, technical communication. Below is your project framed in that specific format, focusing on the "Architecture" and "Design" of your simulator.

# ---

**Technical Spec: Vintage Amp Repair Sim (VARS)**

**Author(s):** \[Your Name\]

**Status:** Draft

**Last Updated:** 2024-05-22

## **1\. Overview**

VARS is a web-based educational game designed to teach the fundamentals of 1970s-era discrete electronics repair. The system simulates the electrical behavior of analog circuits, allowing users to diagnose and "repair" hardware using a mobile-first interface.

## **2\. Goals & Non-Goals**

### **Goals**

* **Logic-to-Hardware Bridge:** Translate CS concepts (logic gates, state machines) into physical components (transistors, capacitors).  
* **Physical Accuracy:** Use Nodal Analysis to ensure voltages and currents behave realistically.  
* **Mobile-First Accessibility:** Optimized for touch interactions (dragging probes, scrubbing potentiometers).  
* **Feedback Loop:** Provide real-time audio and visual feedback via the Web Audio API.

### **Non-Goals**

* **Full CAD Suite:** We are not building a generic circuit simulator (like SPICE). Circuits are pre-defined templates.  
* **3D Graphics:** The game will use 2D/2.5D visual representations to ensure performance on mobile browsers.

## **3\. Context**

Modern electronics are often "black boxes" (integrated circuits). 1970s gear is "discrete," meaning every part is visible and replaceable. For a CS student, this provides a unique opportunity to see the "Physical Layer" of the OSI model in a literal sense.

## **4\. Proposed Design**

### **A. The Engine (The "Kernel")**

The backend logic will treat the circuit as a **Directed Graph**.

* **Nodes:** Junction points in the circuit.  
* **Edges:** Components (Resistors, Capacitors, Inductors).  
* **Solver:** A MatrixSolver class implementing **Modified Nodal Analysis (MNA)**.  
  * *Formula:* \[G\]\[V\] \= \[I\], where G is the conductance matrix, V is node voltages, and I is current sources.

### **B. Component Data Model**

Every component is an object extending a base Component class:

TypeScript

interface Component {  
  id: string;  
  nominalValue: number; // e.g., 100uF  
  actualValue: number;  // Can "drift" due to age  
  failureMode: 'nominal' | 'open' | 'shorted' | 'leaky';  
  position: { x: number, y: number };  
}

### **C. The Interaction Layer**

* **The Multimeter Tool:** A UI overlay that calculates the potential difference between two selected nodes in the MatrixSolver.  
* **The "DeoxIT" Mechanic:** A touch-event handler that reduces the resistance property of variable resistors (knobs) based on "scrubbing" velocity.

## **5\. Implementation Plan**

### **Phase 1: The Solver (MVP)**

* Build a basic JS-based matrix solver.  
* Input: A JSON file representing a simple voltage divider.  
* Output: Correct voltage readings at specific nodes.

### **Phase 2: Visual Canvas**

* Use HTML5 Canvas to render a 2D top-down view of a PCB (Printed Circuit Board).  
* Implement "Long-Press to Desolder" and "Drag to Replace."

### **Phase 3: Audio Integration**

* Map the output node of the circuit simulation to a ScriptProcessorNode in the Web Audio API.  
* If a capacitor is "leaky," apply a low-pass filter or distortion effect to the audio stream.

## **6\. Detailed Design: Circuit Graph**

To maintain mobile performance, the circuit graph will be solved only when the user "probes" a point or changes a component. This prevents high CPU usage on mobile devices compared to real-time SPICE simulations.

## **7\. Security & Privacy**

* **Local Storage:** Save game progress and "inventory" of spare parts locally.  
* **No Backend Required:** The game should be fully functional offline as a Progressive Web App (PWA).

## **8\. Open Questions**

* **Performance:** Can a mobile browser handle a matrix solver with \>50 nodes? (Potential solution: Use Web Workers).  
* **Assets:** How to procedurally generate "dust" and "corrosion" textures for the vintage aesthetic?

### ---

**How to use this with Spec-Kit:**

1. **Clone** github/spec-kit.  
2. **Copy** the content above into a new .md file in the specs/ directory.  
3. **Use the PR template** provided in that repo to submit your "Design Doc" to yourself or collaborators to refine the logic before you write your first line of React/p5.js code.

Since you're a CS major, would you like to see how the **Matrix Solver** (MNA) would look in TypeScript?