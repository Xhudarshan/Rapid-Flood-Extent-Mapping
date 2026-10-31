"""FloodSight Terai V2 — offline demo dashboard; not live flood monitoring."""
import csv
import io
import json
from datetime import datetime, timezone

import streamlit as st

st.set_page_config(page_title="FloodSight Terai V2", page_icon="🌊", layout="wide")

DEMO = [
    ("Susta", "Nawalparasi West", 27.515, 83.765, 8200, 1640, 0.91),
    ("Tribeni", "Nawalparasi West", 27.465, 83.910, 6100, 1190, 0.76),
    ("Bardaghat", "Nawalparasi West", 27.535, 83.665, 15400, 3180, 0.52),
    ("Rajbiraj", "Saptari", 26.540, 86.750, 28500, 5750, 0.63),
    ("Kanchanpur", "Saptari", 26.620, 86.790, 9800, 1910, 0.84),
    ("Hanumannagar", "Saptari", 26.575, 86.930, 7200, 1400, 0.88),
    ("Nepalgunj Rural", "Banke", 28.050, 81.610, 13200, 2650, 0.69),
    ("Rapti Sonari", "Banke", 27.980, 81.760, 11400, 2210, 0.79),
    ("Kohalpur", "Banke", 28.210, 81.690, 21800, 4490, 0.38),
    ("Tikapur", "Kailali", 28.528, 81.118, 25500, 5120, 0.73),
    ("Bhajani", "Kailali", 28.390, 81.080, 10800, 2100, 0.86),
    ("Lamki", "Kailali", 28.420, 81.180, 17600, 3560, 0.47),
]
FIELDS = ["settlement", "district", "lat", "lon", "population", "buildings", "water_exposure"]

def parse_csv(file):
    raw = file.getvalue().decode("utf-8-sig")
    reader = csv.DictReader(io.StringIO(raw))
    if not reader.fieldnames or not set(FIELDS).issubset(reader.fieldnames):
        raise ValueError("CSV must include: " + ", ".join(FIELDS))
    result = []
    for line, row in enumerate(reader, 2):
        try:
            name = row["settlement"].strip()
            district = row["district"].strip()
            lat, lon = float(row["lat"]), float(row["lon"])
            pop, buildings = int(row["population"]), int(row["buildings"])
            exposure = float(row["water_exposure"])
            if not name or not district or not (-90 <= lat <= 90) or not (-180 <= lon <= 180):
                raise ValueError("invalid location")
            if pop < 0 or buildings < 0 or not (0 <= exposure <= 1):
                raise ValueError("counts must be nonnegative and exposure must be 0–1")
            result.append((name, district, lat, lon, pop, buildings, exposure))
        except (ValueError, TypeError, AttributeError) as exc:
            raise ValueError(f"Invalid data on CSV line {line}: {exc}") from exc
    if not result:
        raise ValueError("CSV has no data rows")
    return result

def analyze(rows, threshold, spread):
    output = []
    for name, district, lat, lon, pop, buildings, exposure in rows:
        fraction = max(0.0, min(1.0, exposure * spread - threshold))
        people = round(pop * fraction)
        homes = round(buildings * fraction)
        priority = "HIGH" if fraction >= 0.55 else "MEDIUM" if fraction >= 0.25 else "LOW" if fraction > 0 else "MONITOR"
        output.append({
            "Settlement": name, "District": district, "lat": lat, "lon": lon,
            "Exposure score (demo)": exposure,
            "Estimated affected people": people,
            "Estimated affected buildings": homes,
            "Flood fraction (demo)": round(fraction, 3), "Priority": priority,
        })
    return sorted(output, key=lambda r: (r["Estimated affected people"], r["Flood fraction (demo)"]), reverse=True)

def csv_export(rows):
    out = io.StringIO()
    writer = csv.DictWriter(out, fieldnames=list(rows[0]))
    writer.writeheader()
    writer.writerows(rows)
    return out.getvalue().encode("utf-8")

st.title("🌊 FloodSight Terai")
st.caption("Nakshatra 2026 · Team Spartans · Nepal Terai")
st.warning("DEMONSTRATION ONLY: The built-in exposure values are synthetic, not satellite observations. Do not use for emergency decisions.")

with st.sidebar:
    st.header("Scenario controls")
    threshold = st.slider("Flood threshold (demo)", 0.0, 0.8, 0.25, 0.05)
    spread = st.slider("Water spread multiplier", 0.5, 1.5, 1.0, 0.05)
    uploaded = st.file_uploader("Optional: settlement CSV", type=["csv"])
    st.caption("Columns: " + ", ".join(FIELDS))

rows = DEMO
if uploaded is not None:
    try:
        rows = parse_csv(uploaded)
        st.success(f"Loaded {len(rows)} settlements from CSV")
    except ValueError as exc:
        st.error(str(exc) + ". Showing sample data instead.")

results = analyze(rows, threshold, spread)
people = sum(r["Estimated affected people"] for r in results)
buildings = sum(r["Estimated affected buildings"] for r in results)
high = sum(r["Priority"] == "HIGH" for r in results)

c1, c2, c3, c4 = st.columns(4)
c1.metric("Settlements", len(results))
c2.metric("People potentially affected", f"{people:,}")
c3.metric("Buildings potentially affected", f"{buildings:,}")``
c4.metric("High priority", high)

t1, t2, t3 = st.tabs(["🗺️ Map", "📋 Impact table", "📥 Export"])
with t1:
    st.subheader("Settlement locations")
    st.map([{"lat": r["lat"], "lon": r["lon"]} for r in results])
    st.caption("Pins show settlement locations, not detected flood polygons.")
with t2:
    st.subheader("Scenario-based settlement ranking")
    st.dataframe(results, hide_index=True, use_container_width=True)
    st.caption("Priorities are illustrative and are not official rescue rankings.")
with t3:
    st.subheader("Download demonstration results")
    st.download_button("Download CSV", csv_export(results), "floodsight_demo.csv", "text/csv")
    report = {"mode": "SYNTHETIC DEMO — NOT FOR EMERGENCY USE",
              "generated_utc": datetime.now(timezone.utc).isoformat(),
              "threshold": threshold, "spread_multiplier": spread,
              "settlements": results}
    st.download_button("Download JSON", json.dumps(report, indent=2), "floodsight_demo.json", "application/json")

st.info("Roadmap: add actual Sentinel-1 pre/post SAR processing, Otsu change detection, permanent-water filtering, U-Net inference, validated population overlays and flood polygons.")
