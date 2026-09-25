# Audio Heatmap Visualizer (Obstacle Detection)

Simulates a linear microphone array picking up an echo reflected off an obstacle, then reconstructs where the obstacle is using delay-and-sum (DAS) beamforming — the same underlying idea behind sonar, radar, and ultrasound B-mode imaging.

## Running

```bash
python "Obstacle Detection with Audio.py"
```

Requires `numpy` and `matplotlib`. It pops up 4 plots in sequence (each blocks until closed): the simulated per-microphone waveforms, a heatmap of that raw data, the reconstructed obstacle-location image for a single simulated obstacle, and a second reconstruction from an embedded real multi-obstacle dataset. Edit the `params = setup(...)` blocks to change the mic array size/spacing, source position, wave speed, or obstacle location.

## The theory, as I understand it

**Forward model (simulating what the mics hear):** A source at a fixed point emits a short pulse, modeled here as a `sinc(SincP * t)` function (a sinc pulse is a reasonable stand-in for a short, band-limited "click," since its Fourier transform is a clean rectangular frequency band). That pulse travels out, reflects off an obstacle at some point, and arrives at each microphone in a linear array. Each mic hears a delayed copy of the same pulse, where the delay is proportional to the total path length `source → obstacle → mic` divided by the wave speed `C`. Since the mics are laid out in a line and the obstacle is off to one side, each mic is a slightly different distance from the obstacle, so they each get a slightly different delay — mics closer to the obstacle hear the echo sooner. Plotting all mic signals stacked by position (`waveforms.png`) or as a mic-vs-time heatmap (`raw_heatmap.png`) shows this directly: the echo traces out a curved (hyperbola-like) band across the mic array, because delay is a nonlinear (square-root, from the distance formula) function of mic position.

**Inverse problem (delay-and-sum beamforming):** Given only the recorded signals, `find_obstacle()` tries to reconstruct where the reflection actually came from, without knowing the true obstacle position in advance. For every candidate point in a 2D grid, it computes what the round-trip delay *would have been* from the source to that candidate point to each mic, looks up the recorded sample at that delay for every mic, and sums them. If the candidate point is close to the real obstacle, the predicted delays line up with where the real echo actually landed in each mic's recording, so the summed samples add constructively (all near a peak) — a bright value. If the candidate point is wrong, the predicted delays are off for most mics, so the samples being summed are essentially uncorrelated noise-like values from different parts of each waveform, and they mostly cancel out — a dim value. Repeating this over the whole grid produces an image (`data2`) where brightness at each point is roughly "how consistent is this location with the observed echo," and the brightest point is the detected obstacle location (marked with a red dot in `reconstructed_single.png`).

This is exactly delay-and-sum beamforming: the same principle used in phased-array radar/sonar and medical ultrasound imaging, just in a small simulated 2D setup with a straight mic array instead of a curved probe.

**Point-spread artifacts:** Notice the reconstructed image isn't a clean dot — there's a smeared, bowtie/hourglass-shaped region of elevated brightness around the true obstacle. This is the array's point-spread function: with a finite number of mics over a finite aperture (line length), delay-and-sum can't perfectly distinguish a real obstacle from other points that happen to produce a very similar set of delays across the array, so nearby/along-axis points also light up somewhat. More mics, wider spacing (larger aperture), or a sharper pulse (higher `SincP`) would tighten this up, at various practical tradeoffs (e.g. spacing wider than half a wavelength introduces aliasing/ghost images, exactly like an FFT interpretation of the sinc pulse's frequency content).

**Multiple obstacles:** `find_obstacle(..., multiple_obstacles=True)` runs the same reconstruction but instead of only marking the single brightest point, it marks every point within a small threshold of the maximum — which is how `reconstructed_multi.png` (using a real embedded multi-mic dataset, not the simulated single-obstacle case) ends up with several obstacles located in one image.

## Screenshots

**Simulated per-mic waveforms** (source emits a sinc pulse, each mic sees a delayed copy):
![waveforms](waveforms.png)

**Same raw data as a mic-vs-sample heatmap** (the curved echo trace across mics):
![raw heatmap](raw_heatmap.png)

**Delay-and-sum reconstruction, single simulated obstacle at (3, -1):**
![reconstructed single](reconstructed_single.png)

**Delay-and-sum reconstruction from a real multi-obstacle dataset:**
![reconstructed multi](reconstructed_multi.png)
