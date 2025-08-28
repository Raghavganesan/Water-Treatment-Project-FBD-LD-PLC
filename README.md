<h1>Water Treatment Tank – 3‑Pump PLC Control</h1>

<div class="badge">CODESYS / Ladder + ST / FBD</div>
<div class="badge">Hydrostatic Level 0–10 m</div>
<div class="badge">Fault Logging + Auto Reset Window</div>
<p class="muted" style="color:#c2cbe0">Robust PLC logic to operate three distribution pumps based on tank level, with overload isolation, fault counters, and sensor scaling.</p>
</div>

<section class="card" id="overview">
<h2>1) Overview</h2>
  <img width="1351" height="777" alt="Screenshot 2025-07-12 143722" src="https://github.com/user-attachments/assets/0d50c188-3f2f-47d8-9bf7-1f489f75fe8d" />

<p>
A water treatment project uses a <strong>10&nbsp;m</strong> water deposit (tank) to supply a village via <strong>three pumps</strong> (P1, P2, P3). A
<strong>hydrostatic analog level sensor</strong> measures tank level. Pumps engage progressively as the water level rises, and disengage when it falls below their thresholds. Each pump has an independent <em>overload</em> input that trips only the affected pump. Faults are counted per pump and periodically reset.
</p>
</section>


<section class="card" id="requirements">
<h2>2) Functional Requirements</h2>
<ul>
<li><strong>Start/Stop</strong>: One global <em>Start</em> and one global <em>Stop</em> push‑button.</li>
<li><strong>Level bands</strong> (in meters):
<ul>
<li><strong>&lt; 1&nbsp;m</strong>: No pump runs.</li>
<li><strong>2–5&nbsp;m</strong>: Run <strong>P1</strong>.</li>
<li><strong>6–8&nbsp;m</strong>: Run <strong>P1 + P2</strong>.</li>
<li><strong>9–10&nbsp;m</strong>: Run <strong>P1 + P2 + P3</strong>.</li>
</ul>
</li>
<li><strong>Overloads</strong>: Each pump has an overload digital input. When active, only that pump stops; the others continue if demanded.</li>
<li><strong>Fault counters</strong>: Increment per pump on each overload event.</li>
<li><strong>Reset window</strong>: Fault counters auto‑reset every <strong>3 days</strong> (simulation mode: every <strong>1 minute</strong>).</li>
<li><strong>Analog scaling</strong>: Sensor raw signal <code>400…20000</code> (counts) → level <code>0…10&nbsp;m</code>.</li>
<li><strong>Displayed level</strong>: meters only (no centimeters).</li>
</ul>
</section>


<section class="card" id="bands">
<h2>4) Level Bands &amp; Pump Logic</h2>
<table>
<thead>
<tr><th>Level (m)</th><th>Demanded Pumps</th><th>Comments</th></tr>
</thead>
<tbody>
<tr><td>&lt; 1</td><td>—</td><td>Dry‑run protection: all pumps off</td></tr>
<tr><td>2 – 5</td><td>P1</td><td>Distribution begins</td></tr>
<tr><td>6 – 8</td><td>P1 + P2</td><td>Medium demand</td></tr>
<tr><td>9 – 10</td><td>P1 + P2 + P3</td><td>Peak demand / high level</td></tr>
</tbody>
</table>
<p class="note">Use small hysteresis (e.g., ±0.1&nbsp;m) around each boundary to avoid relay chatter.</p>
</section>

<section class="card" id="scaling">
<h2>5) Analog Scaling (Raw → Meters)</h2>
 

<img width="799" height="432" alt="image" src="https://github.com/user-attachments/assets/c759dc04-fdb2-4e98-ae29-caee8c911bfd" />

<p>
Convert raw counts <code>x</code> from <code>400…20000</code> to level in meters <code>L</code> for the <code>0…10&nbsp;m</code> span:
</p>
<pre><code>L = clamp( 0, 10,
(x - 400) * (10.0 / (20000 - 400)) )
// Use integer math with rounding if required by the PLC platform.</code></pre>
<p>Round or floor to the nearest <strong>0.1&nbsp;m</strong> for display; internal logic can use the full precision.</p>
</section>

<section class="card" id="interlocks">
<h2>6) Interlocks &amp; Overload Behaviour</h2>
<ul>
<li><strong>Start/Stop latch</strong>: System runs only after <code>Start</code>; any <code>Stop</code> drops all pump commands.</li>
<li><strong>Per‑pump overload</strong>: If <code>DI_OL_Pi</code> is active, force <code>DO_Pi_Run</code> <span class="err">OFF</span> and increment <em>Pi_FaultCount</em>. Other pumps remain unaffected.</li>
<li><strong>Dry level interlock</strong>: If <code>L &lt; 1 m</code>, force all pumps <span class="err">OFF</span>.</li>
<li><strong>Priority</strong>: P1 &gt; P2 &gt; P3 (always engage lower‑index pumps first).</li>
</ul>
</section>


<section class="card" id="logging">
<h2>7) Fault Logging &amp; Reset Policy</h2>
<ul>
<li>Maintain <code>UINT</code> counters: <code>P1_Faults</code>, <code>P2_Faults</code>, <code>P3_Faults</code>.</li>
<li>Increment on the rising edge of each overload signal.</li>
<li>Auto‑reset window: every <strong>3 days</strong> using a time accumulator or RTC comparison.
<ul>
<li><em>Simulation mode</em>: reset every <strong>1 minute</strong> (parameter‑selectable).</li>
</ul>
</li>
<li>Optional: expose counters on HMI and keep the last reset timestamp.</li>
</ul>
</section>


<section class="card" id="simulation">
<h2>9) Simulation &amp; Testing</h2>
  <img width="1623" height="367" alt="image" src="https://github.com/user-attachments/assets/f40f0a60-2dd2-40fd-ad87-0d39f9855c83" />

  <img width="1784" height="793" alt="image" src="https://github.com/user-attachments/assets/5852a323-4f37-46c4-b871-cab5a0026916" />

<ol>
<li>Set <strong>Simulation Mode</strong> parameter to enable <em>1‑minute</em> reset window.</li>
<li>Ramp <code>AI_LevelRaw</code> from 400 → 20000 and observe band transitions and output commands.</li>
<li>Trigger each <code>DI_OL_Pi</code> while demanded to verify pump isolation and fault counter increments.</li>
<li>Drop <code>Level_m</code> below 1&nbsp;m to confirm dry‑level shutdown of all pumps.</li>
<li>Verify counters auto‑reset after the window; check the last reset timestamp (if enabled).</li>
</ol>
</section>
