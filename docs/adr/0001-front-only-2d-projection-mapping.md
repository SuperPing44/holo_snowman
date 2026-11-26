# Front-only 2D projection mapping with one projector

The Statue is viewed from the front only and lit by a single projector placed roughly at viewer position. Visitors draw on a 2D front-view template, and each Surface's region of that template is warped onto the projector image by a per-Surface 2D calibration (ellipses for Balls, four-corner shapes for Panels). We chose this over 3D mapping (a virtual Statue model with a calibrated projector camera) because the project is built by one person in about nine weeks for a single event (Tag der offenen Tür, 29.–30.01.2027), and from the front the 2D result is visually close to the 3D one.

## Consequences

- The sides and back of the Statue stay unlit; the edges of the Balls show slight stretching.
- Moving the projector or the Statue means recalibrating in operator mode.
- Adding wrap-around or multi-projector projection later would be a rewrite of the mapping layer, not an extension of it.
