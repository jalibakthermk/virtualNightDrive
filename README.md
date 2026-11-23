# virtualNightDrive

A night-driving view lit per pixel from headlamp IES files. One self-contained page, `index.html`: open it in a browser
with WebGL 2 (it loads Three.js r128 and the shared [beamKit](https://jalibakthermk.github.io/beamkit/demo.html) styles and fonts from the web, so it needs internet).

## Inputs

### Display
| Control | What it does |
|---|---|
| Night view / Illuminance (lx) | Eye view of the scene, or false-colour horizontal illuminance |
| Driver eye / Bird's eye | Camera position |
| No cockpit / Car / Truck | Cabin silhouette over the driver view |
| Adapt to rendered view / Road estimate | Source of the eye adaptation luminance (see below) |
| Auto exposure from eye adaptation | Ward (1994) contrast-based exposure from the adaptation luminance |
| Exposure trim (EV) | Offset on top of the automatic exposure |
| Shadows | Shadows of trees, posts and signs |
| Distance markers, marker distances | Overlay lines at the listed distances (m, comma separated) |
| Isolux lines, isolux levels | Contours of road illuminance (lx, comma separated) |

### Photometry
| Control | What it does |
|---|---|
| Halogen / HID / LED, colour temperature | Modelled source spectrum: light colour, S/P ratio |
| Eye colour adaptation | CAT02 adaptation towards the source white (dark surround) |
| Mesopic vision | CIE 191:2010 mesopic brightness and colour fade |
| IES files (low/high beam, left/right lamp) | Type A, B or C; one file per function is enough |
| Vertical aim offset | Shared vertical aim of both lamps |
| Mirror H axis | Flips loaded files left to right |

### Vehicle
| Control | Default |
|---|---|
| Lamp mounting height, lamp spacing | 0.70 m, 1.20 m |
| Eye height, eye behind lamps, eye lateral offset | 1.20 m, 2.00 m, -0.37 m (offside) |
| Windscreen in the driver view, transmission | On, 0.80 (placeholder, see below) |

### Road
| Control | Default |
|---|---|
| RHT / LHT, road width | RHT, 7.0 m |
| Flat road | Off: the track has crests and sags |
| Dry / Wet, road surface R_L, marking R_L | Dry, 13 and 150 mcd/m²/lx (wet: 3 and 35) |
| Road brightness by eye height | On |
| Sign brightness by observation angle, sheeting class | On, RA2 |

### ADB scenario (from adbSim)
| Control | What it does |
|---|---|
| Load adbSim road scenario | The file adbSim exports under Project and export, Road-illuminate scenario (format `adbsim-road-scenario`) |
| Play, scenario time | Run the scenario in real time (or slower), or step to any time |
| Traffic: oncoming cars and trucks | Show the scenario's vehicles; without a scenario, oncoming traffic on the track |

The file embeds the low beam and the ADB segments (one IES file per segment, or one high beam with segment boundaries)
and, per time frame, each lamp's segment weights and the target vehicles. Each lamp's high beam is rebuilt per frame as
the sum of w_i × segment_i, so you see the dark tunnel move as the controller in adbSim switched it. While a scenario is
loaded the road is straight and flat, as adbSim assumes; lamp height, spacing, vertical aim, traffic side and lane width
come from the file, and the car drives at the scenario's ego speed. The overlay lists the segments that are off and the
illuminance at each target's glare point with adbSim's limit result. Unload the scenario to return to the track and your
own files.

### Driving
Set speed, lane keeping, steering with A/D or the on-screen arrows. Keys: W/S speed, H high beam, C lux view,
V camera, M markers, Space pause. When paused, Save PNG exports the view with a caption of the settings.

## Models

**Traffic.** Cars (4.5 × 1.8 m) and trucks (16.5 × 2.55 m) with headlamps and tail lamps at the heights and spacing of
adbSim's target types. Bodies are lit by your beams like any other object; their lamps are drawn self-luminous with a
soft glow and are not part of the photometric calculation.

**Retroreflection of signs.** For each lamp, at each sign pixel, the observation angle (lamp and driver eye seen from
the sign) and entrance angle give R_A from the selected sheeting table. Luminance is
L = Σ R_A(α, β) · E⊥ / cos v, with E⊥ = I/d² and v the viewing angle. The driver's real eye is used in every view.
The panel shows the observation angle at 100 m for the current geometry.

**Eye adaptation.** A 64 × 32 luminance image (cd/m², before exposure) is rendered from the driver's eye every
4th frame, 2° below the horizon with a 24° vertical field. The adaptation luminance is its log-average with Gaussian
centre weighting (σ 10° across, 4° up and down); sky pixels are left out. Float render target where the device supports
it, otherwise 16-bit log encoding in an 8-bit target. The road estimate (flat, straight road sampled on the CPU) is
the fallback.

**Vertical profile.** Parabolic vertical curves joining ±4 % grades, two crests and three sags per 1,502 m track
period, net elevation zero. The vehicle pitches with the grade; lamps, eye and beam lookup follow it. While the
profile is on, "Road illuminance ahead" and the road estimate are flat-road values and are labelled so.

**Road surface.** Road and markings use R_L (L = R_L · E⊥) plus a diffuse term. R_L is defined at the EN 1436 30 m car
geometry; with "Road brightness by eye height" on it is scaled per lamp and per point by G / 0.54, where
G = tan(lamp elevation) / tan(eye elevation) seen from the road point (Sørensen 2011, Annex B). A higher eye, as in a
truck, sees a darker road surface. Wet mode lowers R_L and adds mirror
reflections of the roadside objects: a camera mirrored in the road plane renders their luminance, and the road shader
projects into that image with water's Fresnel reflectance and a vertical spread. Reflections need float render
targets; without them the wet road only darkens.

**Windscreen.** Scene luminance is multiplied by the transmission before adaptation and tone mapping, in the driver
view only.

## Reference data

| Value | Used for | Source |
|---|---|---|
| R_A tables RA1, RA2 (α 0.2°, 0.33°, 2°; β 5°, 30°, 40°) | Sign sheeting | EN 12899-1:2007 Tables 3 and 4 (white, minimum values) |
| R_A table RA3 | Sign sheeting | **Placeholder**, not standard data |
| R_A beyond β 40° falls with cos β / cos 40° | Sign sheeting | **Assumption**, the standard gives no values |
| Marking R_L 150 dry, 35 wet | Road markings | EN 1436 classes R3 and RW2 (minimum values) |
| Road surface R_L 13 dry, 3 wet | Asphalt | Measured means of bare pavement, EN 1436 geometry (*Road marking visibility for automated vehicles*, Case Studies in Construction Materials, 2025, Table 7). Typical asphalt and concrete: dry 6–35, wet 3–8 (BASt report V 235, Drewes et al., 2014) |
| R_L scaling with viewing geometry, G = 0.54 at 30 m | Road and markings | K. Sørensen, *Performance of road markings and road surfaces* (2011), Annex B |
| Verge R_L 4 | Grass verge | **Placeholder** |
| Wet mirror share 0.5 | Wet reflections | **Placeholder** |
| Water reflectance, F0 = 0.02 (n = 1.33) | Wet reflections | Fresnel equations, Schlick approximation |
| Crest K 26, sag K 30, grades ±4 % | Vertical profile | AASHTO Green Book (2018), 80 km/h design values |
| Windscreen transmission 0.80 | Driver view | **Placeholder**; legal minimum 0.70 (UN R43, FMVSS 205) |
| Mesopic model | Brightness and colour | CIE 191:2010 |
| Exposure | Display | Ward (1994) contrast-based scale factor |

Replace the placeholders with measured data for your materials where it matters.
