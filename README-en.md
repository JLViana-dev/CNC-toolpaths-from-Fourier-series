# CNC toolpaths from Fourier series

A single-file web app (`corte-fourier.html`) that describes cutting contours as Fourier series and generates G-code for thermal cutting (plasma, laser, oxy-fuel) and for contour milling, in the dialects of common controllers (Fanuc, Haas, Mitsubishi, Okuma, Heidenhain and LinuxCNC/Mach), as well as tables for manual marking. It runs directly in the browser, including on a phone, with nothing to install, and your files are never uploaded anywhere.

Published version: https://claude.ai/artifact/EAeQpz3ANa6oh22K4eLeVR

Note: the app interface is in Portuguese. Field names below are given in English, with the on-screen Portuguese label in parentheses.

## The idea

A closed contour is a periodic curve, so it can be written as two Fourier series:

```
x(t) = a0 + Σ (aₖ cos kt + bₖ sin kt)
y(t) = c0 + Σ (cₖ cos kt + dₖ sin kt),   k = 1 … N
```

With few terms (small N) the curve is rough. The larger N is, the closer it gets to the original contour. The app shows the maximum error in millimetres, so you can pick the smallest N that meets your tolerance.

## Tab 1: 2D shape (Forma 2D)

For plates and parts with free-form contours.

### Contour input

- draw with a finger or mouse;
- pick a preset (star, heart, gear);
- import a PNG/JPG image (tap the dashed area, drag and drop, or paste with Ctrl+V);
- import a DXF file.

**Image:** the app detects the background from the borders, takes the largest shape, and traces the outer contour and the enclosed holes (up to 8). If it picks the wrong side, tick **Invert** (*Inverter*).

**DXF:** polylines, circles, and chained lines and arcs are read. The largest contour becomes the part, and contours inside it become holes. Units are assumed to be mm, and the part width is filled in automatically. Splines and blocks are not read.

### Parameters

| Field | What it does |
|---|---|
| Part width, mm (*Largura da peça*) | Sets the real-world scale. For images, enter it yourself. |
| Tolerance, mm (*Tolerância*) | Used by the **Automatic N** button (*N automático*). |
| Terms N (*Termos N*) | Number of series terms (1 to 100). |
| Kerf or cutter Ø, mm (*Kerf ou Ø fresa*) | Thermal kerf width or cutter diameter. The path is offset by half of this value. |
| The contour is (*O contorno é*) | Part (cut on the outside) or hole (cut on the inside). Applies to a single contour. |
| Lead-in/out, mm (*Entrada/saída*) | Length of the lead-in and lead-out. With 0, there is none. |
| Controller (*Controlador*) | G-code dialect to generate (see the table below). |
| Process (*Processo*) | Thermal cutting (on/off with `M3`/`M5`) or contour milling (plunge and retract in Z, with spindle speed). |
| Depth, step, safe Z, speed S | Milling only: total depth, depth per pass, clearance plane and spindle speed. |
| Feed F, mm/min (*Avanço F*) | Cutting feed rate. Half of it is used for the Z plunge. |
| Program number (*Nº do programa*) | Used in `O0001` and in the name `CORTE1`. |

**Several contours:** when there are holes, the **Active contour** selector (*Contorno ativo*) and the **Cut** checkbox (*Cortar*) let you include or exclude each one. The part gets the kerf on the outside and each hole on the inside, automatically.

**Output:** the **Generate G-code** button (*Gerar G-code*) writes the program in the text box, and **Copy** (*Copiar*) or **Download** (*Baixar*, file `corte-gcode.txt`) takes the text to your machine.

### How the G-code is built

- Units in mm, absolute coordinates (`G21 G90`) and the feed rate `F` you enter.
- The curve is sampled at 720 points per contour.
- The kerf is already applied to the path. **Do not use cutter compensation (G41/G42) on the machine.**
- Order: holes first, part last.
- With lead-in/out greater than 0: the cut starts outside the part, on the scrap side, and reaches the contour along a perpendicular straight line. At the end, the cut overlaps a short stretch (at least 2 mm or the kerf) and exits through the scrap.
- In **thermal cutting**, each contour turns the cut on with `M3` and off with `M5`. Adapt these commands to your controller.
- In **contour milling**, each contour rises to safe Z, moves to the entry point, plunges (at half feed), cuts and returns to safe Z. If the depth is greater than the step, it repeats in several passes.
- The program starts with a comment asking for validation in a simulator and giving the total path length and estimated time.

### Controllers

| Controller | Start | End | Notes |
|---|---|---|---|
| Generic ISO | `G21 G90 G17` | `M30` | Base for the others. |
| Fanuc | `%`, `O0001 (CORTE1)`, `G21 G90 G17 G40 G49 G80`, `G54` | `M30`, `%` | |
| Haas | Same as Fanuc, with a 5-digit program number (`O00001`) | In milling: `G28 G91 Z0`, `G90`, `M30`, `%` | |
| Mitsubishi | `O0001`, `G21 G90 G17 G40 G80`, `G54` | `M30` | Fanuc-compatible mode. |
| Okuma | `O0001`, `G21 G90 G17 G40 G80`, `G15 H1` | `M02` | |
| Heidenhain | `BEGIN PGM CORTE1 MM`, `BLK FORM` and `TOOL CALL` (milling) | `M30`, `END PGM CORTE1 MM` | Klartext with numbered lines and `L ... R0` moves. |
| LinuxCNC / Mach3 | `G21 G90 G17 G40 G64 P0.05` | `M30` | Common in plasma cutting. |

**Important:** the headers and footers are templates based on common conventions of each controller family. They have not been validated on machines or in the manufacturers' simulators. Check your controller's manual and simulate before use.

### Kerf compensation

In milling, the same field holds the cutter diameter. The offset curve is computed with the normal obtained from the analytic derivatives of the series itself:

```
P' = P + (kerf/2) · n
```

If the kerf is larger than the radius of curvature at some point (common at sharp corners), the path forms loops and the app warns you. Increase N or reduce the kerf.

## Tab 2: Pipe to pipe (Tubo com tubo)

Cutting profile of a branch pipe for fitting onto a main pipe.

**Inputs:** main pipe diameter D, branch pipe diameter d (d ≤ D), and the angle between the axes, from 20° to 90°.

With R = D/2, r = d/2 and α the angle between the axes, the cut height for the angular position φ around the branch is:

```
s(φ) = [ √(R² − r² sin²φ) − r cos φ cos α ] / sin α
```

Since s(φ) is periodic, it is approximated by a Fourier series with N terms (1 to 12). The app shows:

- the pipe's unrolled plot (exact curve in grey, series in colour);
- the maximum error in mm;
- the branch perimeter (πd) and the cut height;
- a table of heights every 15°, measured from the lowest point of the cut, for marking the pipe by hand.

**Angle reference:** 0° is the obtuse-angle side and 180° is the acute-angle side.

## Limitations

- Wall thickness and root gap are not part of the pipe calculation.
- At sharp corners the series shows the Gibbs phenomenon: the error falls slowly with N. The app shows this error in mm, so use the tolerance as a guide.
- For images, only the largest contour and its holes are read, not several separate parts.
- Lead-in and lead-out are straight lines, not arcs.
- The order between holes is not optimized.
- Milling covers only the contour of the outer shape or of holes already defined: there are no pockets, drilling or tool changes, and no `G41`/`G42` (the offset is geometric).
- The default feed and speed values are examples. Adjust them to the material and tool.
- There is no output for Siemens Sinumerik, nor post-processors for lathes.
- The downloaded file comes out as `.txt`, because the platform does not accept other extensions. Rename it to the extension your controller expects.
- The import, kerf, lead-in/out and hole features were only checked for code syntax and with a simulated browser, not with real files or machines. Test with your own files and check the G-code in a simulator before cutting material.

## Code structure

Everything is in a single `corte-fourier.html`, with no external dependencies:

- `build`, `ev`, `evd`: series fitting (by DFT, with uniform arc-length sampling), evaluation and derivatives.
- `trace`, `comps`, `fromImg`: contour extraction from images.
- `parseDxf`: DXF reading.
- `path`: kerf offset and normals.
- `post`: post-processor (builds the header, moves and ending for each controller).
- `prof`, `fs`: pipe profile and its series.

## Reference

The idea of rebuilding a shape with Fourier series comes from animations showing two epicycle circles (one for x(t) and one for y(t)) tracing a contour, such as the Koch curve, a violin and an octopus.
