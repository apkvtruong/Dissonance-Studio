# Dissonance Studio

An interactive app for deriving consonant intervals and chords from sound spectra, using the psychoacoustic model of sensory dissonance developed by Plomp, Levelt, and Sethares.

Given any sound spectrum (set of partials), Dissonance Studio computes which intervals and triads sound most consonant—and the results depend entirely on the timbre. Harmonic timbres produce the familiar Western intervals; inharmonic timbres (bells, metallophones) produce entirely different scales. Everything updates live while you edit, and every point on every plot can be heard.

The main version is **`dissonance_v1.2.py`**.

---

## Features

- **Three voices (A, B, C)**, each with its own spectrum: pick a preset, mirror another voice ("Same as A"), or type partials and amplitudes.
- **Draggable spectrum editor**: drag a partial sideways to retune it, up or down to re-weight it (Shift: amplitude only, Ctrl: snap to a whole harmonic). Double-click to add a partial, right-click to remove one.
- **Three linked views**
  - **Curve**: dissonance of A with B swept across the ratio range, with consonant minima labelled.
  - **Surface**: 3D OpenGL dissonance surface over every B and C ratio (flat or shaded).
  - **Heatmap**: the same triad space from above, with iso-contours.
- **Ranked results**: the most consonant intervals (by dissonance or by ratio) and triads (within one octave or all voicings). Hovering a row highlights it on the plots, and vice versa.
- **Hear everything**: click any plot point or list entry to play it; press and drag on the curve or heatmap to glide the pitch smoothly. Playback is click-free, with crossfades and low-latency output.
- **Inspector**: ratio, cents, nearest 12-TET interval, nearest just ratio (within 12¢), dissonance, and the number of partials the voices share.
- **Overtone view**: the partials of the last sound on one log-frequency axis, with coinciding partials ringed.
- **Manual playback** of a note, an interval or a triad at exact ratios.
- **Export**: publication figures (PDF, SVG, PNG) of all three views through a live, true-size figure editor, and 24-bit WAV files of any sound.
- **Live analysis**: compiled, multi-threaded Numba kernels recompute while you type or drag; slow jobs are cancelled as soon as a newer edit arrives.
- Dark and light themes (the light theme uses the colour-blind-safe Okabe–Ito palette).

### Controls

| Action | Input |
|--------|-------|
| Hear a point | Click a plot or a result |
| Glide | Press and drag on the curve or heatmap |
| Pan / zoom | Shift-drag / scroll wheel |
| Rotate the surface | Drag on the Surface view |
| Re-analyse and reset views | Ctrl+Enter |
| Replay the last sound | Space |
| Stop audio | Esc |

---

## The Research

### Why Do Some Note Combinations Sound Good?

In 1965, Dutch researchers **Reinier Plomp** and **Willem Levelt** conducted experiments measuring how pleasant or unpleasant pairs of pure tones (sine waves) sounded when played together. They found that dissonance (roughness) between two pure tones depends on their frequency separation relative to the **critical bandwidth** of the auditory system—a frequency-dependent range determined by the physical structure of the inner ear (the basilar membrane).

> R. Plomp and W.J.M. Levelt, "Tonal Consonance and Critical Bandwidth," *Journal of the Acoustical Society of America*, 38, 548–560 (1965).

In 1993, **William Sethares** extended this work by showing that the consonance of complex tones (real musical sounds with multiple partials) can be predicted by summing the dissonance contributions of every pair of partials. Crucially, he demonstrated that **musical scales are not universal—they emerge from timbre**. The Western 12-tone equal temperament scale is well-suited to harmonic timbres (strings, winds), but instruments with inharmonic spectra (bells, gamelans) are naturally consonant with entirely different scales.

> W.A. Sethares, "Local Consonance and the Relationship Between Timbre and Scale," *Journal of the Acoustical Society of America*, 94(3), 1218–1228 (1993).
>
> W.A. Sethares, *Tuning, Timbre, Spectrum, Scale*, Springer (1998/2005).

---

## The Math

### Step 1: Dissonance Between Two Pure Tones

The Plomp-Levelt model gives the sensory dissonance $d$ between two sine tones at frequencies $f_1$ and $f_2$ with loudnesses $\ell_1$ and $\ell_2$:

$$
d(f_1, f_2, \ell_1, \ell_2) = \ell_{12} \left( e^{-b_1 p} - e^{-b_2 p} \right)
$$

where:

- $\ell_{12} = \min(\ell_1, \ell_2)$ — dissonance is limited by the quieter tone
- $p = s \cdot (f_{\max} - f_{\min})$ — frequency difference scaled to the critical bandwidth
- $s = \dfrac{x}{s_1 \cdot f_{\min} + s_2}$ — critical bandwidth scaling factor

The constants, fit to Plomp and Levelt's experimental data by Sethares:

| Parameter | Value | Meaning |
|-----------|-------|---------|
| $b_1$ | 3.51 | Controls the rise of dissonance |
| $b_2$ | 5.75 | Controls the fall of dissonance |
| $x$ | 0.24 | Peak dissonance at ~24% of critical bandwidth |
| $s_1$ | 0.0207 | Critical bandwidth slope |
| $s_2$ | 18.96 | Critical bandwidth offset |

The function rises quickly from zero (unison) to a peak near 25% of the critical bandwidth, then falls off gradually. Two tones very close together "beat" and sound rough; tones far apart don't interact.

### Step 2: Loudness from Amplitude

Amplitudes are converted to a perceptual loudness scale:

$$
\ell = \frac{2^{(\text{dB}/10)}}{16}, \qquad \text{dB} = 20 \log_{10}(a)
$$

This approximates Stevens' power law for loudness perception.

### Step 3: Dissonance of Complex Tones (Intervals)

A real musical sound is a spectrum of partials: a fundamental frequency $f_0$ multiplied by frequency ratios $r_1, r_2, \ldots, r_n$ with amplitudes $a_1, a_2, \ldots, a_n$.

To find which **intervals** sound consonant, we fix one voice (voice A at pitch ratio 1) and sweep a second voice (voice B at pitch ratio $c$) from 0.9 up to **MAX RATIO** (2.3 by default) in steps of 0.002. The total dissonance at ratio $c$ is the sum over all pairs of partials between and within the two voices:

$$
D(c) = \frac{1}{2} \sum_{i,j} \Big[ d(f_i^A,\, f_j^A) + d(c \cdot f_i^B,\, c \cdot f_j^B) + 2\, d(f_i^A,\, c \cdot f_j^B) \Big]
$$

where $f_i^A = f_0 \cdot r_i$ are the partial frequencies of voice A, and similarly for B.

The first term is the self-dissonance of voice A (constant), the second is the self-dissonance of voice B (varies with $c$ because the Plomp-Levelt model is frequency-dependent), and the third is the cross-dissonance between voices.

The curve is normalised so its peak is 1. **Local minima** in $D(c)$ correspond to consonant intervals—pitch ratios where partials tend to align rather than clash.

### Step 4: Dissonance of Triads (3D Surface)

For three voices A, B and C at pitch ratios $1$, $r$ and $s$ (each voice may have its own spectrum), the total dissonance sums every pair of partials in the combined chord—within each voice and between each pair of voices:

$$
D(r, s) = \tfrac{1}{2}\big[S_A + S_B(r) + S_C(s)\big] + X_{AB}(r) + X_{AC}(s) + X_{BC}(r, s)
$$

where $S_V = \sum_{i,j} d(f_i^V, f_j^V)$ is the self-dissonance of a voice and $X_{UV} = \sum_{i,j} d(f_i^U, f_j^V)$ the cross-dissonance between two voices, each evaluated at that voice's ratio.

$r$ and $s$ both run from 1 to MAX RATIO in steps of **3D STEP** (0.005 by default; the grid is capped at 1600 × 1600 points), and the surface is normalised so its peak is 1. **Valleys** in this surface are consonant triads—three-note chords with minimal roughness.

### Step 5: Finding Consonant Intervals and Triads

- **Intervals**: a point is a minimum if it is the lowest within ±5 samples, and it is kept only if it sits at least 0.002 (on the normalised scale) below the lower of the two peaks that separate it from lower ground.
- **Triads**: every grid point lower than its eight neighbours seeds a pattern search on the exact model in $(\ln r, \ln s)$, moving along the six directions in which A–B, A–C and B–C partial coincidences form valleys. This places each triad off the grid and merges the ripples a grid leaves along narrow valleys. Minima that leave the grid are dropped, minima within 2 cents of each other are merged, and triads in which two voices share (almost) the same pitch are removed. Results are ranked by dissonance.

---

## What the Results Show

For a **harmonic spectrum** (partials at 1×, 2×, 3×, 4×, 5×, 6× the fundamental), the dissonance curve minima land on the familiar just intonation intervals:

| Ratio | Interval | Cents |
|-------|----------|-------|
| 1:1 | Unison | 0 |
| 6:5 | Minor third | 316 |
| 5:4 | Major third | 386 |
| 4:3 | Perfect fourth | 498 |
| 3:2 | Perfect fifth | 702 |
| 5:3 | Major sixth | 884 |
| 2:1 | Octave | 1200 |

Other spectra produce different intervals. With only odd harmonics (pan flute, clarinet-like), minima appear at ratios such as 1.400 (7:5) and 1.800 (9:5) that a harmonic tone doesn't favour. An ideal glockenspiel bar (partials at 1, 2.758, 5.406, 8.936, …) gives minima at 1.272, 1.396, 1.654 and 1.790—none of them a Western interval, and its curve has no minimum at the octave at all.

This is the core insight of Sethares' work: **scales and timbre are fundamentally linked**.

---

## Spectrum Presets

| Preset | Partial ratios | Amplitudes | Notes |
|--------|---------------|------------|-------|
| Harmonic | 1, 2, 3, 4, 5, 6 | 1/n | Ideal sawtooth — produces the Western intervals |
| Pan Flute (Odd Harmonics) | 1, 3, 5, 7, 9, 11 | 1/n | Odd partials as in Sethares' measured pan flute; clarinets are similar |
| Glockenspiel (Ideal Bar) | 1, 2.758, 5.406, 8.936, 13.35, 18.645 | 0.88ⁿ | Free–free Euler–Bernoulli bar |
| Bonang | 1, 1.52, 3.46, 3.92 | 0.88ⁿ | Sethares' generic Javanese bonang kettle |
| Saron (Swastigitha) | 1, 2.76, 4.72, 5.92 | 0.88ⁿ | Sethares' generic saron for gamelan Swastigitha |
| Saron (Kyai Kaduk Manis) | 1, 2.39, 2.78, 4.75, 5.08, 5.96 | 0.88ⁿ | Saron with split partial pairs near 2.7 and 4.8 |

The 0.88ⁿ amplitudes are Sethares' generic roll-off, not measurements. Hover a preset in the app for its source. Voices B and C can also mirror another voice ("Same as A") or use any custom spectrum.

---

## Running Dissonance Studio

> **Note:** this section will be updated soon, once the Windows app has been recompiled.

### Windows app (no Python needed)

Download `DissonanceStudio.zip` from the [Releases](../../releases) page, unzip it anywhere and run `DissonanceStudio.exe`. Keep the `.exe` next to its `_internal` folder. The first launch compiles the analysis engine (a few seconds); later launches reuse it.

Windows may show a SmartScreen warning because the app isn't signed: choose **More info → Run anyway**.

### From source

Requires Python 3.12 (64-bit) and a graphics driver with OpenGL for the 3D view.

```bash
pip install -r requirements.txt
python dissonance_v1.2.py
```

The first run compiles the Numba kernels and caches them in `__pycache__`; later runs start in about a second. Without Numba the app falls back to plain NumPy, which is much slower.

### Analysis settings

| Setting | Default | Meaning |
|---------|---------|---------|
| FUNDAMENTAL Hz | 261.63 | Pitch of voice A (middle C) |
| MAX RATIO | 2.3 | Upper end of the curve and of both triad axes |
| 3D STEP | 0.005 | Grid spacing of the triad surface; smaller is finer but slower |

With **Live** on, the analysis reruns automatically as you edit; **Analyse** (Ctrl+Enter) recomputes everything and resets the views.

---

## Other Versions

The repository keeps the earlier versions for reference:

| File | Description |
|------|-------------|
| `dissonance_v1.2.py` | **The current version** |
| `dissonance_v1.0.py` | First release (some features missing) |
| `dissonance_calculation.py` | Original calculation script with matplotlib plots |
| `dissonance_prototype.py` | First prototype with a basic UI |

---

## References

1. R. Plomp and W.J.M. Levelt, "Tonal Consonance and Critical Bandwidth," *J. Acoust. Soc. Am.*, 38, 548–560 (1965).
2. W.A. Sethares, "Local Consonance and the Relationship Between Timbre and Scale," *J. Acoust. Soc. Am.*, 94(3), 1218–1228 (1993).
3. W.A. Sethares, *Tuning, Timbre, Spectrum, Scale*, 2nd ed., Springer (2005).
4. H. Helmholtz, *On the Sensations of Tone*, Dover (1877/1954).
