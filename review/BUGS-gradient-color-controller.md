# Bug review: gradient-color-controller

Repo: EssentialBlocks/gradient-color-controller (public). Fork: suhaeel/gradient-color-controller
Branch: `review/gradient-color-controller-bugs` (no changes to code, main untouched)
What it is: gradient picker control for the background controls: `index.js` (173 lines), `helper.js` (36 lines, the gradient string parser), `constants.js`.

## What I ran
`parseGradientColor()` from `helper.js` was run in Node on 12 gradient strings, including the format the control itself writes. The control calls it with the saved value every time it is opened.

| Input | Result |
|---|---|
| `linear-gradient(45deg, #ff0000 0% , #0000ff 100%)` (its own output) | correct |
| `radial-gradient(ellipse at 50% 50%, #ff0000 0% , #0000ff 100%)` | correct |
| `linear-gradient(90deg, #000000 10% , rgba(255,255,255,0.5) 90%)` | correct |
| `linear-gradient(45deg, red 0% , blue 100%)` | **both colours become `#0000` (transparent)** |
| `linear-gradient(45deg, hsl(0,100%,50%) 0% , blue 100%)` | both colours transparent, and positions come out as 100 and 50 (it picks the `%` inside `hsl()`) |
| `rgba(0, 0, 0, 0.5)` (with spaces) | colour not recognised, so the second colour moves up to the first slot and the second becomes `undefined` |
| `rgba(255,255,255,0.755)` (3 decimals, more than 16 characters) | same as above |
| `linear-gradient(to right, #000 0%, #fff 100%)` | **throws** `Cannot read properties of null (reading '0')` |
| `radial-gradient(circle at 50% 50%, #000 0%, #fff 100%)` (no space before the comma) | first position is `null`, so it becomes `NaN%` |
| empty string `""` | **throws** |
| `undefined` | **throws** `Cannot read properties of undefined (reading 'match')` |

## Findings
| # | Severity | Where | What | Evidence |
|---|---|---|---|---|
| 1 | **High** | helper.js:9-11 | Only hex and `rgb()/rgba()` are recognised. A gradient written with named colours, `hsl()` or `transparent`, or with spaces in `rgba()`, is read back with transparent or missing colours. | **Verified (ran it)** (table above) |
| 2 | **High** | index.js:19-50 | The parsed values are put into state, and a second effect calls `onChange(...)` with them. So opening the control **rewrites the saved gradient** with whatever the parser understood. For a named-colour gradient that means the saved value is replaced by a transparent one without the user touching anything. On first render the second effect also runs once with the initial `transparent` defaults before the parsed values arrive. | Parser output **Verified (ran it)**; effect order From reading the code, not run |
| 3 | **High** | helper.js:15,19,31 | A gradient with no `deg` (`to right`), an empty string, or an undefined value makes the parser throw. The control is in the inspector, so the exception takes the inspector panel down. `propTypes` says the value is required, but nothing guarantees it. | **Verified (ran it)** |
| 4 | Medium | helper.js:21-25 | The radial branch needs the exact `% ,` spacing that the control itself writes, so a radial gradient from another source gives `NaN`. | **Verified (ran it)** |
| 5 | Medium | helper.js:23-24 | `(?<=at )` and `(?<=% )` are regex lookbehinds. Safari before 16.4 cannot parse them, which makes the whole script a syntax error. Safari 16.4 came out in March 2023, so it only matters for old devices. | From reading the code, not run |
| 6 | Low | index.js:134 | `Angle` and positions have `min=0`, so `-45deg` style angles are not possible. | From reading the code, not run |

## Not covered
- Unless a row says "Verified", I only read the code. No automated tests were run.
- Chrome-only and WordPress 7.1.3 / PHP 8.2 on Local where something was run.

## Fix branch
Branch `fix/robust-gradient-parsing`, pushed to the fork `suhaeel/gradient-color-controller`. Based on the default branch; the default branch itself is untouched.

Rewrites `parseGradientColor` to split the arguments at top-level commas and keep each colour as written, never throws, and returns the control's defaults for an empty or unreadable value. The control now starts from the saved value and reports only changes made by the user.

Tested: all 12 inputs from the table above parse without throwing and keep named, hsl and spaced rgba colours; 2000 random round trips through the control's own output format with 0 mismatches. **Rendered the component with React and jsdom**: opening it with `red -> green` called `onChange` twice on master (first with a transparent gradient, then with `#0000 #0000`); on the fix branch it makes **0 calls**, and picking a new first colour keeps the second one (`blue 0%, green 100%`).
Not done: a gradient written with `to right` still shows angle 45 in the controls, because the control has no direction field.
