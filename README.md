# I-m-your-friend

<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>All-in-One Voice App (Background Mode)</title>
  <script src="https://unpkg.com/peerjs@1.5.2/dist/peerjs.min.js"></script>
  <style>
    * { box-sizing: border-box; font-family: system-ui, -apple-system, sans-serif; }
    body {
      margin: 0; padding: 16px;
      background: #0f172a; color: #f8fafc;
      display: flex; flex-direction: column; align-items: center;
    }
    .card {
      background: #1e293b; border-radius: 16px;
      padding: 20px; width: 100%; max-width: 400px;
      box-shadow: 0 10px 25px rgba(0,0,0,0.4);
      margin-bottom: 16px; text-align: center;
    }
    h1 { font-size: 1.3rem; margin: 0 0 8px 0; color: #38bdf8; }
    p { font-size: 0.85rem; color: #94a3b8; margin: 0 0 14px 0; }
    .badge {
      display: inline-block; background: #065f46; color: #6ee7b7;
      font-size: 0.75rem; padding: 4px 10px; border-radius: 20px;
      margin-bottom: 12px; display: none;
    }
    .code-box {
      font-size: 1.8rem; font-weight: bold; letter-spacing: 4px;
      background: #0f172a; color: #4ade80;
      padding: 10px; border-radius: 10px; margin: 12px 0;
      border: 1px dashed #4ade80; display: none;
    }
    input {
      width: 100%; padding: 12px; font-size: 1.05rem;
      border-radius: 10px; border: 1px solid #334155;
      background: #0f172a; color: #fff; text-align: center;
      margin-bottom: 10px;
    }
    button {
      width: 100%; padding: 13px; font-size: 0.95rem; font-weight: 600;
      border: none; border-radius: 10px; cursor: pointer; margin-top: 6px;
    }
    .btn-create { background: #ef4444; color: white; }
    .btn-join { background: #3b82f6; color: white; }
    .btn-mute { background: #eab308; color: #0f172a; display: none; }
    .btn-rec { background: #10b981; color: white; display: none; }
    .btn-stop { background: #64748b; color: white; display: none; }
    .status {
      margin-top: 12px; font-size: 0.85rem;
      padding: 8px 10px; border-radius: 8px;
      background: #0f172a; color: #cbd5e1;
    }
    a.download-link {
      display: block; margin-top: 10px; color: #38bdf8;
      font-size: 0.9rem; text-decoration: underline;
    }
  </style>
</head>
<body>

  <div class="card">
    <h1>🎙️🎧 Live Voice (Background Ready)</h1>
    <div id="bgBadge" class="badge">🔒 Wake-Lock & Background Audio Active</div>
    <p>Create or join a room below. Keep Chrome unrestricted in battery settings for background use.</p>

    <button id="createBtn" class="btn-create" onclick="startRoom()">1. Create Room (Get Code)</button>
    <div id="roomDisplay" class="code-box">----</div>

    <hr style="border: 0; border-top: 1px solid #334155; margin: 16px 0;">

    <input type="number" id="joinCodeInput" placeholder="Enter 4-Digit Room Code" />
    <button id="joinBtn" class="btn-join" onclick="joinRoom()">2. Join Room (Connect)</button>

    <button id="muteBtn" class="btn-mute" onclick="toggleMute()">Mute My Mic</button>
    <button id="recBtn" class="btn-rec" onclick="toggleRecording()">⏺️ Record Incoming Voice</button>
    <button id="stopBtn" class="btn-stop" onclick="location.reload()">Disconnect</button>

    <div id="statusText" class="status">Status: Ready</div>
    <div id="downloads"></div>
    <audio id="remoteAudio" autoplay playsinline></audio>
  </div>

  <script>
    let peer = null;
    let localStream = null;
    let remoteStreamGlobal = null;
    let mediaRecorder = null;
    let recordedChunks = [];
    let isMuted = false;
    let isRecording = false;
    let wakeLock = null;
    const PREFIX = "allinone-voice-";

    async function enableBackgroundMode() {
      document.getElementById('bgBadge').style.display = 'inline-block';
      try {
        if ('wakeLock' in navigator) {
          wakeLock = await navigator.wakeLock.request('screen');
        }
      } catch (e) {}

      if ('mediaSession' in navigator) {
        navigator.mediaSession.metadata = new MediaMetadata({
          title: 'Live Voice Active',
          artist: 'All-in-One Voice App',
          album: 'Background Stream'
        });
        navigator.mediaSession.playbackState = 'playing';
      }
    }

    document.addEventListener('visibilitychange', async () => {
      if (wakeLock !== null && document.visibilityState === 'visible') {
        try { wakeLock = await navigator.wakeLock.request('screen'); } catch (e) {}
      }
    });

    async function getMic() {
      if (!localStream) {
        localStream = await navigator.mediaDevices.getUserMedia({
          audio: { echoCancellation: true, noiseSuppression: true, autoGainControl: true },
          video: false
        });
      }
      await enableBackgroundMode();
      return localStream;
    }

    async function startRoom() {
      const status = document.getElementById('statusText');
      try {
        status.innerText = "Allow microphone access...";
        await getMic();
        const code = Math.floor(1000 + Math.random() * 9000);
        peer = new Peer(PREFIX + code);

        peer.on('open', () => {
          document.getElementById('roomDisplay').style.display = 'block';
          document.getElementById('roomDisplay').innerText = code;
          status.innerHTML = "✅ Room Live! Share code <b>" + code + "</b>.";
          showControls();
        });

        peer.on('call', (call) => {
          call.answer(localStream);
          handleCall(call);
        });
      } catch (e) {
        status.innerText = "Error: Please open over HTTPS and allow Mic.";
      }
    }

    async function joinRoom() {
      const code = document.getElementById('joinCodeInput').value.trim();
      const status = document.getElementById('statusText');
      if (code.length !== 4) {
        status.innerText = "Enter a 4-digit code first.";
        return;
      }
      try {
        status.innerText = "Allow microphone access...";
        await getMic();
        status.innerText = "Connecting to " + code + "...";
        peer = new Peer();

        peer.on('open', () => {
          const call = peer.call(PREFIX + code, localStream);
          handleCall(call);
        });
      } catch (e) {
        status.innerText = "Error: Please open over HTTPS and allow Mic.";
      }
    }

    function handleCall(call) {
      call.on('stream', (remoteStream) => {
        remoteStreamGlobal = remoteStream;
        const audio = document.getElementById('remoteAudio');
        audio.srcObject = remoteStream;
        audio.play();
        enableBackgroundMode();
        document.getElementById('statusText').innerText = "🔊 Connected! Working in foreground & background.";
        document.getElementById('recBtn').style.display = 'block';
        showControls();
      });
    }

    function showControls() {
      document.getElementById('createBtn').style.display = 'none';
      document.getElementById('joinBtn').style.display = 'none';
      document.getElementById('joinCodeInput').style.display = 'none';
      document.getElementById('muteBtn').style.display = 'block';
      document.getElementById('stopBtn').style.display = 'block';
    }

    function toggleMute() {
      if (!localStream) return;
      isMuted = !isMuted;
      localStream.getAudioTracks()[0].enabled = !isMuted;
      document.getElementById('muteBtn').innerText = isMuted ? "Unmute My Mic (Listening Only)" : "Mute My Mic";
    }

    function toggleRecording() {
      const recBtn = document.getElementById('recBtn');
      if (!isRecording) {
        recordedChunks = [];
        mediaRecorder = new MediaRecorder(remoteStreamGlobal);
        mediaRecorder.ondataavailable = (e) => { if (e.data.size > 0) recordedChunks.push(e.data); };
        mediaRecorder.onstop = saveRecording;
        mediaRecorder.start();
        isRecording = true;
        recBtn.innerText = "⏹️ Stop & Save Recording";
        recBtn.style.background = "#ef4444";
      } else {
        mediaRecorder.stop();
        isRecording = false;
        recBtn.innerText = "⏺️ Record Incoming Voice";
        recBtn.style.background = "#10b981";
      }
    }

    function saveRecording() {
      const blob = new Blob(recordedChunks, { type: 'audio/webm' });
      const url = URL.createObjectURL(blob);
      const a = document.createElement('a');
      a.href = url;
      a.download = 'voice-recording-' + Date.now() + '.webm';
      a.className = 'download-link';
      a.innerText = "⬇️ Tap to Download Saved Voice Clip";
      document.getElementById('downloads').appendChild(a);
    }
  </script>
</body>
</html>
