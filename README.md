# Quantum-Key-Security-Lab-Simulating-BB84-with-Eavesdropping-and-Noise
# Import libraries
from qiskit import QuantumCircuit
from qiskit_aer import Aer
import numpy as np
import matplotlib.pyplot as plt
from random import choice
import tkinter as tk
from tkinter import ttk, messagebox
import csv
from datetime import datetime

# BB84 Key Exchange (No Eavesdropping)
def bb84_key_exchange(n_bits, loss_rate=0.0):
    alice_bits = [choice([0, 1]) for _ in range(n_bits)]
    alice_bases = [choice(['+', 'x']) for _ in range(n_bits)]
    circuits = []
    for i in range(n_bits):
        qc = QuantumCircuit(1, 1)
        if alice_bits[i] == 1:
            qc.x(0)
        if alice_bases[i] == 'x':
            qc.h(0)
        qc.measure(0, 0)
        circuits.append(qc)
    
    bob_bases = [choice(['+', 'x']) for _ in range(n_bits)]
    backend = Aer.get_backend('qasm_simulator')
    bob_results = []
    for i in range(n_bits):
        if np.random.random() < loss_rate:
            bob_results.append(None)
            continue
        if bob_bases[i] == 'x':
            circuits[i].h(0)
        circuits[i].measure(0, 0)
        job = backend.run(circuits[i], shots=1, memory=True)
        result = job.result().get_memory()[0]
        bob_results.append(int(result))
    
    sifted_key_alice, sifted_key_bob = [], []
    for a_bit, a_base, b_bit, b_base in zip(alice_bits, alice_bases, bob_results, bob_bases):
        if b_bit is not None and a_base == b_base:
            sifted_key_alice.append(a_bit)
            sifted_key_bob.append(b_bit)
    
    errors = sum(1 for a, b in zip(sifted_key_alice, sifted_key_bob) if a != b)
    qber = errors / len(sifted_key_alice) if sifted_key_alice else 0
    return sifted_key_alice, sifted_key_bob, qber

# BB84 with Eavesdropping (PNS or Intercept-Resend)
def bb84_with_eavesdropping(n_bits, attack_type="PNS", p_multi_photon=0.2, loss_rate=0.0):
    alice_bits = [choice([0, 1]) for _ in range(n_bits)]
    alice_bases = [choice(['+', 'x']) for _ in range(n_bits)]
    eve_knowledge = []
    circuits = []
    
    backend = Aer.get_backend('qasm_simulator')
    
    for i in range(n_bits):
        qc = QuantumCircuit(1, 1)
        if alice_bits[i] == 1:
            qc.x(0)
        if alice_bases[i] == 'x':
            qc.h(0)
        
        if attack_type == "PNS" and np.random.random() < p_multi_photon:
            eve_knowledge.append(alice_bits[i])
        elif attack_type == "Intercept-Resend":
            eve_base = choice(['+', 'x'])
            if eve_base == 'x':
                qc.h(0)
            qc.measure(0, 0)
            job = backend.run(qc, shots=1, memory=True)
            eve_bit = int(job.result().get_memory()[0])
            eve_knowledge.append(eve_bit)
            qc = QuantumCircuit(1, 1)
            if eve_bit == 1:
                qc.x(0)
            if eve_base == 'x':
                qc.h(0)
        
        qc.measure(0, 0)
        circuits.append(qc)
    
    bob_bases = [choice(['+', 'x']) for _ in range(n_bits)]
    bob_results = []
    for i in range(n_bits):
        if np.random.random() < loss_rate:
            bob_results.append(None)
            continue
        if bob_bases[i] == 'x':
            circuits[i].h(0)
        circuits[i].measure(0, 0)
        job = backend.run(circuits[i], shots=1, memory=True)
        result = job.result().get_memory()[0]
        bob_results.append(int(result))
    
    sifted_key_alice, sifted_key_bob = [], []
    for a_bit, a_base, b_bit, b_base in zip(alice_bits, alice_bases, bob_results, bob_bases):
        if b_bit is not None and a_base == b_base:
            sifted_key_alice.append(a_bit)
            sifted_key_bob.append(b_bit)
    
    errors = sum(1 for a, b in zip(sifted_key_alice, sifted_key_bob) if a != b)
    qber = errors / len(sifted_key_alice) if sifted_key_alice else 0
    eve_info_rate = len(eve_knowledge) / n_bits
    return sifted_key_alice, sifted_key_bob, qber, eve_info_rate

# Privacy Amplification
def privacy_amplification(key, reduction_factor=0.5):
    new_length = int(len(key) * reduction_factor)
    return key[:new_length]

# Secure Key Rate (Simplified)
def secure_key_rate(qber):
    if qber >= 0.11:
        return 0
    h = -qber * np.log2(qber) - (1 - qber) * np.log2(1 - qber) if 0 < qber < 1 else 0
    return 1 - h

# Run Experiments with Varying Attack Strength
def run_experiments(n_bits, trials, attack_type, p_multi_photon_values, loss_rate):
    results = []
    for p_multi_photon in p_multi_photon_values:
        qber_no_eve = []
        qber_with_eve = []
        eve_info_rates = []
        
        for _ in range(trials):
            _, _, qber = bb84_key_exchange(n_bits, loss_rate)
            qber_no_eve.append(qber)
            _, _, qber_eve, eve_info_rate = bb84_with_eavesdropping(n_bits, attack_type, p_multi_photon, loss_rate)
            qber_with_eve.append(qber_eve)
            eve_info_rates.append(eve_info_rate)
        
        results.append({
            'p_multi_photon': p_multi_photon,
            'avg_qber_no_eve': np.mean(qber_no_eve),
            'avg_qber_with_eve': np.mean(qber_with_eve),
            'avg_eve_info_rate': np.mean(eve_info_rates),
            'secure_key_rate': secure_key_rate(np.mean(qber_with_eve))
        })
    return results

# UI and Main Logic
class QKDApp:
    def __init__(self, root):
        self.root = root
        self.root.title("Quantum Cryptography Security Simulator")
        
        # Labels and Inputs
        tk.Label(root, text="Number of Bits:").grid(row=0, column=0, padx=5, pady=5)
        self.n_bits = tk.Entry(root)
        self.n_bits.insert(0, "100")
        self.n_bits.grid(row=0, column=1)
        
        tk.Label(root, text="Number of Trials:").grid(row=1, column=0, padx=5, pady=5)
        self.trials = tk.Entry(root)
        self.trials.insert(0, "10")
        self.trials.grid(row=1, column=1)
        
        tk.Label(root, text="Attack Type:").grid(row=2, column=0, padx=5, pady=5)
        self.attack_type = ttk.Combobox(root, values=["None", "PNS", "Intercept-Resend"])
        self.attack_type.set("PNS")
        self.attack_type.grid(row=2, column=1)
        
        tk.Label(root, text="PNS Probability (0-1):").grid(row=3, column=0, padx=5, pady=5)
        self.p_multi_photon = tk.Entry(root)
        self.p_multi_photon.insert(0, "0.2")
        self.p_multi_photon.grid(row=3, column=1)
        
        tk.Label(root, text="Photon Loss Rate (0-1):").grid(row=4, column=0, padx=5, pady=5)
        self.loss_rate = tk.Entry(root)
        self.loss_rate.insert(0, "0.1")
        self.loss_rate.grid(row=4, column=1)
        
        # Run Button
        tk.Button(root, text="Run Simulation", command=self.run_simulation).grid(row=5, column=0, columnspan=2, pady=10)
        
        # Output Text (with centered tag)
        self.output = tk.Text(root, height=10, width=50)
        self.output.grid(row=6, column=0, columnspan=2, padx=5, pady=5)
        self.output.tag_configure("center", justify="center")

        # Center the window on the screen
        self.root.update_idletasks()
        window_width = self.root.winfo_reqwidth()
        window_height = self.root.winfo_reqheight()
        screen_width = self.root.winfo_screenwidth()
        screen_height = self.root.winfo_screenheight()
        x = (screen_width // 2) - (window_width // 2)
        y = (screen_height // 2) - (window_height // 2)
        self.root.geometry(f"{window_width}x{window_height}+{x}+{y}")

    def run_simulation(self):
        try:
            n_bits = int(self.n_bits.get())
            trials = int(self.trials.get())
            attack_type = self.attack_type.get()
            p_multi_photon = float(self.p_multi_photon.get())
            loss_rate = float(self.loss_rate.get())
            
            if not (0 <= p_multi_photon <= 1 and 0 <= loss_rate <= 1):
                raise ValueError("Probabilities must be between 0 and 1.")
            
            self.output.delete(1.0, tk.END)
            self.output.insert(tk.END, "Running BB84 Simulation...\n", "center")
            
            # Run experiments with varying attack strength
            p_multi_photon_values = [0.1, 0.2, 0.3, 0.4, 0.5] if attack_type != "None" else [p_multi_photon]
            results = run_experiments(n_bits, trials, attack_type if attack_type != "None" else "PNS", p_multi_photon_values, loss_rate)
            
            # Save results to a CSV file
            timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
            with open(f"qkd_results_{timestamp}.csv", "w", newline='') as f:
                writer = csv.DictWriter(f, fieldnames=['p_multi_photon', 'avg_qber_no_eve', 'avg_qber_with_eve', 'avg_eve_info_rate', 'secure_key_rate'])
                writer.writeheader()
                for result in results:
                    writer.writerow(result)
            
            # Display results
            for result in results:
                self.output.insert(tk.END, f"\nPNS Probability: {result['p_multi_photon']:.2f}\n", "center")
                self.output.insert(tk.END, f"Avg QBER (No Eve): {result['avg_qber_no_eve']:.4f}\n", "center")
                if attack_type != "None":
                    self.output.insert(tk.END, f"Avg QBER (With Eve): {result['avg_qber_with_eve']:.4f}\n", "center")
                    self.output.insert(tk.END, f"Avg Eve Info Rate: {result['avg_eve_info_rate']:.4f}\n", "center")
                self.output.insert(tk.END, f"Secure Key Rate: {result['secure_key_rate']:.4f}\n", "center")
            
            # Sample key
            key_alice, key_bob, qber = bb84_key_exchange(n_bits, loss_rate)
            self.output.insert(tk.END, f"\nSample Key (Alice, first 10): {key_alice[:10]}\n", "center")
            self.output.insert(tk.END, f"Sample Key (Bob, first 10): {key_bob[:10]}\n", "center")
            self.output.insert(tk.END, f"QBER: {qber:.4f}\n", "center")
            amplified_key = privacy_amplification(key_alice)
            self.output.insert(tk.END, f"Amplified Key (first 10): {amplified_key[:10]}\n", "center")
            
            # Plot QBER vs PNS Probability
            if attack_type != "None":
                p_values = [r['p_multi_photon'] for r in results]
                qber_no_eve = [r['avg_qber_no_eve'] for r in results]
                qber_with_eve = [r['avg_qber_with_eve'] for r in results]
                
                plt.figure(figsize=(8, 6))
                plt.plot(p_values, qber_no_eve, label='QBER (No Eve)', marker='o')
                plt.plot(p_values, qber_with_eve, label='QBER (With Eve)', marker='o')
                plt.xlabel('PNS Probability')
                plt.ylabel('Quantum Bit Error Rate (QBER)')
                plt.title('QBER vs PNS Probability')
                plt.legend()
                plt.grid(True)
                plt.show()
            
            # Boxplot for QBER comparison
            qber_no_eve = [r['avg_qber_no_eve'] for r in results]
            qber_with_eve = [r['avg_qber_with_eve'] for r in results]
            plt.boxplot([qber_no_eve, qber_with_eve], labels=['No Eve', f'With {attack_type}'])
            plt.title('QBER Comparison')
            plt.ylabel('Quantum Bit Error Rate')
            plt.show()
            
        except ValueError as e:
            messagebox.showerror("Input Error", str(e))
        except Exception as e:
            messagebox.showerror("Error", f"An error occurred: {str(e)}")

# Main execution
if __name__ == "__main__":
    root = tk.Tk()
    app = QKDApp(root)
    root.mainloop()
