<!doctype html>
<html lang="hi">
<head>
  <meta charset="utf-8"/>
  <meta name="viewport" content="width=device-width,initial-scale=1"/>
  <title>Share location (consent required)</title>
  <style>
    body{font-family:system-ui,Segoe UI,Roboto,Arial;margin:28px}
    .card{max-width:760px;background:#fafafa;padding:20px;border-radius:10px;box-shadow:0 6px 20px rgba(0,0,0,0.06)}
    button{padding:10px 16px;border-radius:8px;font-size:16px}
    pre{white-space:pre-wrap;background:#fff;border:1px solid #eee;padding:12px;border-radius:8px;margin-top:12px}
    small{color:#666}
  </style>
</head>
<body>
  <div class="card">
    <h2>Location share karne ke liye</h2>
    <p>Agar aap allow karoge to hum aapka approximate area (city/state) dekhenge. Aap jab tak allow nahin karte tab tak kuch bhi share nahin hota. <br><small>Hosted on HTTPS required</small></p>

    <p>
      <button id="ask">Share my location</button>
      <button id="ask-now" style="margin-left:8px">Auto-request on load (try)</button>
    </p>

    <p id="status">Status: idle</p>
    <pre id="out"></pre>
  </div>

<script>
const status = document.getElementById('status');
const out = document.getElementById('out');

// Replace with your server endpoint to receive data (HTTPS). 
// If you don't want to post to server, leave as empty string.
const SERVER_ENDPOINT = ""; // e.g. "https://your-server.example/save-location"

function setStatus(t){ status.textContent = 'Status: ' + t; console.log(t); }
function show(o){ out.textContent = JSON.stringify(o, null, 2); }

/** Reverse geocode using Nominatim (OpenStreetMap) - free */
async function reverseGeocode(lat, lon){
  const url = `https://nominatim.openstreetmap.org/reverse?format=json&lat=${encodeURIComponent(lat)}&lon=${encodeURIComponent(lon)}&zoom=18&addressdetails=1`;
  const res = await fetch(url, { headers: { 'Accept': 'application/json' } });
  if (!res.ok) throw new Error('Geocoding failed: ' + res.status);
  return await res.json();
}

async function handleLocation(pos){
  const payload = {
    latitude: pos.coords.latitude,
    longitude: pos.coords.longitude,
    accuracy: pos.coords.accuracy,
    timestamp: pos.timestamp
  };
  setStatus('Location obtained: ' + payload.latitude + ', ' + payload.longitude);
  show(payload);

  try {
    const geo = await reverseGeocode(payload.latitude, payload.longitude);
    const addr = geo.address || {};
    const area = addr.city || addr.town || addr.village || addr.suburb || addr.hamlet || addr.county || addr.state || geo.display_name;
    setStatus('Area: ' + (area || 'Not found'));
    show({ location: payload, area: area, full: geo.display_name });
  } catch (e) {
    setStatus('Reverse geocoding failed: ' + e.message);
    show({ location: payload, error: e.message });
  }

  // Optional: send to your server (if SERVER_ENDPOINT set)
  if (SERVER_ENDPOINT) {
    try {
      const r = await fetch(SERVER_ENDPOINT, {
        method:'POST',
        headers:{ 'Content-Type':'application/json' },
        body: JSON.stringify({location: payload, obtainedAt: Date.now()})
      });
      if (r.ok) setStatus((status.textContent || '') + ' — Sent to server.');
      else setStatus((status.textContent || '') + ' — Server error: ' + r.status);
    } catch(err){
      setStatus((status.textContent || '') + ' — Network error: ' + err.message);
    }
  }
}

function requestLocation(){
  if (!('geolocation' in navigator)) {
    setStatus('Geolocation not supported by this browser.');
    return;
  }
  if (!confirm('Is this your consent to share location with the requester? (You can Cancel to refuse)')) {
    setStatus('User cancelled consent dialog.');
    return;
  }

  setStatus('Requesting browser permission...');
  navigator.geolocation.getCurrentPosition(handleLocation, (err) => {
    switch(err.code){
      case err.PERMISSION_DENIED: setStatus('Permission denied by user.'); break;
      case err.POSITION_UNAVAILABLE: setStatus('Position unavailable.'); break;
      case err.TIMEOUT: setStatus('Timeout while getting location.'); break;
      default: setStatus('Error: ' + err.message);
    }
    show({ error: err.message, code: err.code });
  }, { enableHighAccuracy: true, timeout: 30000, maximumAge: 0 });
}

// Buttons
document.getElementById('ask').addEventListener('click', requestLocation);

document.getElementById('ask-now').addEventListener('click', () => {
  setStatus('Attempting immediate request...');
  requestLocation();
});
</script>
</body>
</html>
