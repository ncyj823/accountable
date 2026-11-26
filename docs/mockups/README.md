# Mockups

Three looks for the MVP's dashboard and pipeline, all following `docs/spec.md` and `docs/BRAND.md`.
Each HTML file holds both screens, one above the other, with the same made-up figures, so the
comparison is about the design and not the data. Beside each one is a full-page screenshot taken
in Google Chrome at 1440 wide.

Open the three PNGs side by side, pick one, and say which. The choice and the reasons then go into
`docs/spec.md`, and the build follows it.

| File | Direction |
| --- | --- |
| `mockup-a-panels` | The spec as written. White cards on the page colour, a floating sidebar, four stat tiles, charts in rows. Calm and conventional. |
| `mockup-b-editorial` | Fewer, larger panels with a lot of air. Navigation across the top, one hero revenue figure, Instrument Serif doing real work in the headings. |
| `mockup-c-operator` | Dense and data-first. A Midnight sidebar, one metric strip of five figures, two large charts side by side, and the whole dashboard plus the board above the fold. |

All three use the same stage colours, the same darkened text colours and the same panel treatment,
so whichever is chosen, the tokens in the spec do not change.

Two things to know when reading them:

- The mockups pull the three fonts from Google Fonts so they render correctly in a browser. The app
  itself bundles them, because it has to work with no internet connection. Open a mockup offline and
  the type falls back; nothing else changes.
- The charts are hand-drawn SVG. The app uses Recharts, as the spec says, so the real charts will
  carry tooltips and will not match these pixel for pixel.
