# modfaust
```python
import json
from collections import defaultdict
import sys

# ---------------------------------------------------
# 1. Waveform mapping helper
# ---------------------------------------------------
def wave_to_faust_fn(wave):
    mapping = {
        "sine": "osc",
        "sawtooth": "sawtooth",
        "triangle": "triangle",
        "square": "square",
        "noise": "no.noise"
    }
    return mapping.get(wave.lower(), wave)

# ---------------------------------------------------
# 2. Faust code templates
# ---------------------------------------------------
TEMPLATES = {
    "oscillator": "{id} = {osc_fn}({freq});",
    "lowpass": "{id} = fi.lowpass6e({input}, {cutoff});",
    "highpass": "{id} = fi.highpass({input}, {cutoff});",
    "gain": "{id} = ({input}) * {gain};",
    "pan": "{id}_l = ({input}) * sqrt(1 - {pan}); {id}_r = ({input}) * sqrt({pan});",
    "mixer": "{id} = {input};",
    "adsr": "{id} = en.adsr({attack}, {decay}, {sustain}, {release}, {gate});"
}

# ---------------------------------------------------
# 3. Combine and normalize multiple inputs
# ---------------------------------------------------
def combine_inputs(inputs):
    if not inputs:
        return "0"
    if len(inputs) == 1:
        return inputs[0]
    summed = " + ".join(inputs)
    return f"(({summed}) / {len(inputs)})"

# ---------------------------------------------------
# 4. Safe modulation expressions
# ---------------------------------------------------
def safe_mod_expr(base, mod_signal, param_name):
    if "cutoff" in param_name.lower():
        depth = 200
    elif "freq" in param_name.lower():
        depth = 5
    else:
        depth = 1
    return f"abs({base} + ({mod_signal} * {depth}))"

# ---------------------------------------------------
# 5. JSON → Faust generator
# ---------------------------------------------------
def json_to_faust(data):
    inputs = defaultdict(list)
    param_mods = defaultdict(list)

    # Parse connections
    for conn in data["connections"]:
        frm, to = conn["from"], conn["to"]
        if "." in to:
            mod_target, param = to.split(".")
            param_mods[mod_target].append((param, frm))
        else:
            inputs[to].append(frm)

    lines = ['import("stdfaust.lib");', ""]

    stereo_signals = {"left": [], "right": []}
    has_left = has_right = has_output = False
    velocity_defined = False

    for mod in data["modules"]:
        m_id = mod["id"]
        m_type = mod["type"]
        if m_type not in TEMPLATES:
            raise ValueError(f"Unknown module type '{m_type}'")

        params = mod.copy()

        # --- Handle MIDI velocity (global) ---
        if mod.get("velocity", False) and not velocity_defined:
            lines.append('velocity = nentry("velocity[midi:vel]", 1, 0, 1, 0.01);')
            velocity_defined = True

        # --- Oscillator ---
        if m_type == "oscillator":
            fn = wave_to_faust_fn(mod.get("wave", "osc"))
            params["osc_fn"] = f"os.{fn}"
            if fn.startswith("no."):
                lines.append(f"{m_id} = {fn};")
                continue

            if mod.get("midi", False):
                params["freq"] = 'hslider("freq[midi:note]", 440, 20, 20000, 1)'
            else:
                params["freq"] = mod.get("freq", 440)

            expr = TEMPLATES[m_type].format(**params)
            if mod.get("velocity", False):
                expr = f"{expr[:-1]} * velocity;"  # multiply by velocity
            lines.append(expr)
            continue

        # --- ADSR (fixed double assignment + proper gate handling) ---
        if m_type == "adsr":
            gate_expr = "button(\"gate\")"
            if mod.get("midi", False):
                gate_expr = 'button("gate[midi:on]")'
            params["gate"] = combine_inputs(inputs.get(m_id, [])) or gate_expr

            # Base ADSR signal
            base_line = f"{m_id}_base = en.adsr({params['attack']}, {params['decay']}, {params['sustain']}, {params['release']}, {params['gate']});"
            lines.append(base_line)

            # Apply velocity if enabled
            if mod.get("velocity", False):
                lines.append(f"{m_id} = {m_id}_base * velocity;")
            else:
                lines.append(f"{m_id} = {m_id}_base;")
            continue

        # --- Handle modulations safely ---
        for (param, src) in param_mods.get(m_id, []):
            base_val = mod.get(param, 0)
            params[param] = safe_mod_expr(base_val, src, param)

        params["input"] = combine_inputs(inputs.get(m_id, []))

        # --- Gain ---
        if m_type == "gain" and mod.get("velocity", False):
            line = TEMPLATES[m_type].format(**params)
            line = f"{line[:-1]} * velocity;"
            lines.append(line)
            continue

        # --- Pan (fixed phantom pan references) ---
        if m_type == "pan":
            line = TEMPLATES[m_type].format(**params)
            lines.append(line)
            stereo_signals["left"].append(f"{m_id}_l")
            stereo_signals["right"].append(f"{m_id}_r")
            continue


        lines.append(TEMPLATES[m_type].format(**params))

    # --- Output connections ---
    for conn in data["connections"]:
        if conn["to"] == "left":
            has_left = True
            stereo_signals["left"].append(conn["from"])
        elif conn["to"] == "right":
            has_right = True
            stereo_signals["right"].append(conn["from"])
        elif conn["to"] == "output":
            has_output = True
            stereo_signals["left"].append(conn["from"])
            stereo_signals["right"].append(conn["from"])

    if (has_left or has_right) and has_output:
        sys.exit("⚠️ Invalid routing: stereo patch defines both 'output' and 'left/right'.")

    if has_left or has_right:
        left_expr = combine_inputs(stereo_signals["left"])
        right_expr = combine_inputs(stereo_signals["right"])
        lines.append(f"\nprocess = {left_expr}, {right_expr};")
    else:
        mono_expr = combine_inputs(stereo_signals["left"])
        lines.append(f"\nprocess = {mono_expr};")

    return "\n".join(lines)

# ---------------------------------------------------
# 6. Run as script
# ---------------------------------------------------
if __name__ == "__main__":
    with open("patch.json") as f:
        data = json.load(f)
    try:
        faust_code = json_to_faust(data)
    except SystemExit as e:
        print(e)
        sys.exit(1)
    with open("patch.dsp", "w") as f:
        f.write(faust_code)
    print("✅ Generated patch.dsp with MIDI velocity support successfully!")

```
