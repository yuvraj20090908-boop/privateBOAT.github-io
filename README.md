<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>privateBOAT: Peer Chat & QR Pairing</title>
    
    <script src="https://unpkg.com/peerjs@1.5.2/dist/peerjs.min.js"></script>
    <script src="https://cdn.rawgit.com/schmich/instascan/master/dist/instascan.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/qrcode@1.5.3/build/qrcode.min.js"></script>
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0-beta3/css/all.min.css">

    <style>
        /* --- Dark Mode Variables (Centralized Theme) --- */
        :root {
            --bg-dark: #1F1F1F;       
            --bg-surface: #292929;     
            --text-primary: #EAEAEA;
            --text-secondary: #B0B0B0;
            --accent-green: #00A884; 
            --icon-color: #B0B0B0;
            --message-input: #3A3A3A;
            --chat-bubble-local: #005C4B; 
            --chat-bubble-remote: #343434;
        }

        /* --- Global Reset & Layout --- */
        body {
            font-family: 'Segoe UI', Arial, sans-serif;
            margin: 0;
            background-color: var(--bg-dark);
            color: var(--text-primary);
            display: flex;
            flex-direction: column;
            height: 100vh;
            overflow: hidden;
        }

        /* --- Header & Status --- */
        .header {
            background-color: var(--bg-surface);
            padding: 15px;
            box-shadow: 0 2px 4px rgba(0, 0, 0, 0.4);
            display: flex;
            justify-content: space-between;
            align-items: center;
            font-size: 1.5em;
            font-weight: bold;
        }
        #status-bar {
            background-color: var(--bg-surface);
            color: var(--text-secondary);
            padding: 5px 15px;
            font-size: 0.85em;
            text-align: center;
            border-bottom: 1px solid #333;
        }

        /* --- Chat Area --- */
        #content-area {
            flex-grow: 1;
            overflow-y: auto;
            display: flex;
            flex-direction: column;
            padding: 10px;
        }
        .message-bubble {
            padding: 10px 14px;
            border-radius: 18px;
            max-width: 85%;
            margin-bottom: 8px;
            word-wrap: break-word;
            font-size: 0.95em;
            position: relative;
        }
        .msg-remote {
            background-color: var(--chat-bubble-remote);
            align-self: flex-start;
            border-bottom-left-radius: 2px;
        }
        .msg-local {
            background-color: var(--chat-bubble-local);
            align-self: flex-end;
            border-bottom-right-radius: 2px;
        }
        
        /* --- Chat Input --- */
        .chat-input-bar {
            display: flex;
            align-items: center;
            padding: 10px 15px;
            background-color: var(--bg-surface);
        }
        .chat-input-bar input {
            flex-grow: 1;
            padding: 10px 15px;
            border: none;
            border-radius: 20px;
            background-color: var(--message-input);
            color: var(--text-primary);
            margin-right: 10px;
            font-size: 1em;
            outline: none;
        }
        .chat-input-bar i { 
            font-size: 1.5em; 
            color: var(--icon-color); 
            cursor: pointer; 
            padding: 5px;
            transition: color 0.2s;
        }
        #send-message-icon { /* Specific style for the send button when enabled */
            color: var(--accent-green);
        }
        .chat-input-bar i:hover { color: var(--text-primary); }

        /* --- Navigation Bar --- */
        .nav-bar {
            display: flex;
            justify-content: space-around;
            padding: 10px 0;
            background-color: var(--bg-surface);
            border-top: 1px solid #333;
        }
        .nav-bar i { 
            color: var(--icon-color); 
            font-size: 1.8em; 
            cursor: pointer; 
            transition: color 0.2s;
        }
        .nav-bar i.active { 
            color: var(--accent-green); 
            filter: drop-shadow(0 0 5px var(--accent-green));
        }

        /* --- QR/Video View Overlays (Optimized for Mobile) --- */
        #qr-view, #video-view {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            background-color: var(--bg-dark);
            z-index: 100;
            display: none;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            padding: 20px;
            box-sizing: border-box;
        }
        #qr-view canvas, #qr-view video {
            width: 90vmin; /* Responsive to viewport size */
            height: 90vmin;
            max-width: 350px;
            max-height: 350px;
            border-radius: 10px;
            box-shadow: 0 0 10px rgba(0, 0, 0, 0.5);
            object-fit: cover;
        }
        .qr-button {
            background-color: var(--accent-green);
            color: white;
            padding: 10px 20px;
            border: none;
            border-radius: 25px;
            margin: 8px;
            cursor: pointer;
            font-size: 1em;
        }
        
        /* Video Call Specific Styles */
        #video-view { justify-content: flex-start; }
        #remote-video {
            width: 100%;
            height: 100%;
            background-color: black;
            border-radius: 0;
            object-fit: cover;
            position: absolute;
            top: 0;
            left: 0;
            z-index: 1;
        }
        #local-video {
            position: absolute;
            bottom: 120px; 
            right: 20px;
            width: 120px;
            height: 90px;
            border-radius: 8px;
            border: 3px solid white;
            z-index: 10; /* Above remote video */
            object-fit: cover;
            transform: scaleX(-1); /* Mirror effect for local video */
        }
        .video-controls {
            position: absolute;
            bottom: 30px;
            z-index: 20;
            display: flex;
            gap: 20px;
        }
        .video-controls i {
            font-size: 2.5em;
            color: white;
            padding: 15px;
            border-radius: 50%;
            background-color: rgba(0, 0, 0, 0.5);
            cursor: pointer;
        }
        #end-call-button { background-color: #D32F2F; color: white; }
    </style>
</head>
<body>
    <div class="header">
        **privateBOAT** <div>
            <i class="fas fa-video" id="start-video-call" title="Start Video Call"></i>
            <i class="fas fa-bars" style="margin-left: 15px;"></i>
        </div>
    </div>

    <div id="status-bar">Connecting to PeerJS...</div>

    <div id="content-area">
        <div id="message-display">
            <p style="text-align: center; color: var(--text-secondary); margin-top: 50px;" id="initial-message">
                Tap the **Connect** icon (🌐) to start a new chat via QR code.
            </p>
        </div>
    </div>
    
    <div class="chat-input-bar">
        <i class="fas fa-paperclip"></i>
        <input type="text" id="chat-message-input" placeholder="Message" disabled>
        <i class="fas fa-paper-plane" id="send-message-icon"></i>
    </div>

    <div class="nav-bar">
        <i class="fas fa-comment active" id="chat-tab"></i>
        <i class="fas fa-qrcode" id="scan-qr-tab"></i>
        <i class="fas fa-globe" id="show-id-tab"></i>
        <i class="fas fa-cog"></i>
    </div>

    <div id="qr-view">
        <h3 style="color: var(--text-primary); margin-bottom: 20px;" id="qr-title">Connect via QR Code</h3>
        
        <video id="scanner-video" style="display:none;"></video>
        <canvas id="qr-code-canvas" style="display:none;"></canvas>

        <p id="qr-instructions" style="color: var(--text-secondary); text-align: center; margin: 20px 0; max-width: 80%;">
            Tap "Start Scanner" to read an ID, or "Show My QR ID" to be scanned.
        </p>

        <button class="qr-button" id="scan-button">Start Scanner</button>
        <button class="qr-button" id="show-my-qr-button">Show My QR ID</button>
        <button class="qr-button" id="cancel-qr-button">Close</button>
    </div>

    <div id="video-view">
        <video id="remote-video" autoplay playsinline></video>
        <video id="local-video" muted autoplay playsinline></video>
        <div class="video-controls">
            <i class="fas fa-phone-slash" id="end-call-button" title="End Call"></i>
            <i class="fas fa-window-restore" id="pip-button" title="Picture-in-Picture"></i>
        </div>
    </div>


    <script>
        // --- 1. GLOBAL STATE & DOM REFERENCES ---
        let peer = null;
        let peerId = null;
        let dataConn = null; // Data connection (chat)
        let mediaCall = null; // Media connection (video/voice)
        let localStream = null;
        
        // DOM Elements
        const statusEl = document.getElementById('status-bar');
        const qrView = document.getElementById('qr-view');
        const scannerVideo = document.getElementById('scanner-video');
        const qrCanvas = document.getElementById('qr-code-canvas');
        const remoteVideo = document.getElementById('remote-video');
        const localVideo = document.getElementById('local-video');
        const messageDisplay = document.getElementById('message-display');
        const chatInput = document.getElementById('chat-message-input');
        const sendIcon = document.getElementById('send-message-icon');
        const navIcons = document.querySelectorAll('.nav-bar i');

        // Instascan setup
        const scanner = new Instascan.Scanner({ video: scannerVideo, scanPeriod: 5 });


        // --- 2. UI UTILITIES (VIEW MANAGEMENT) ---

        function setStatus(text, color = 'var(--text-secondary)') {
            statusEl.textContent = text;
            statusEl.style.color = color;
        }

        function setActiveTab(activeId) {
            navIcons.forEach(icon => {
                icon.classList.remove('active');
            });
            const activeIcon = document.getElementById(activeId);
            if (activeIcon) {
                activeIcon.classList.add('active');
            }
        }

        function showChatView() {
            // Hide all overlays/special views
            qrView.style.display = 'none';
            document.getElementById('video-view').style.display = 'none';

            // Show chat elements
            document.getElementById('content-area').style.display = 'flex';
            document.querySelector('.chat-input-bar').style.display = 'flex';
            
            localVideo.style.display = 'none'; // Hide local video overlay during chat
            scanner.stop(); // Stop scanner if it was running
            setActiveTab('chat-tab');
            
            // Remove initial message if connected
            const initialMsg = document.getElementById('initial-message');
            if (initialMsg && dataConn) initialMsg.remove();
        }

        function showQRView(mode = 'connect') {
            showChatView(); // Hide current chat view elements
            document.getElementById('content-area').style.display = 'none';
            document.querySelector('.chat-input-bar').style.display = 'none';
            qrView.style.display = 'flex';
            setActiveTab('show-id-tab');

            // Reset QR view to default state
            scannerVideo.style.display = 'none';
            qrCanvas.style.display = 'none';
            document.getElementById('scan-button').style.display = 'block';
            document.getElementById('show-my-qr-button').style.display = 'block';

            if (mode === 'scan') {
                startQRScanner();
            } else if (mode === 'show') {
                if (peerId) {
                    generateQRCode(peerId);
                } else {
                    document.getElementById('qr-title').textContent = 'Generating ID...';
                    document.getElementById('qr-instructions').textContent = 'Please wait for PeerJS connection.';
                }
            } else {
                document.getElementById('qr-title').textContent = 'Connect via QR Code';
                document.getElementById('qr-instructions').textContent = 'Tap "Start Scanner" to read an ID, or "Show My QR ID" to be scanned.';
            }
        }

        function showVideoView() {
            showChatView(); // Hide chat elements
            document.getElementById('video-view').style.display = 'flex';
            document.querySelector('.chat-input-bar').style.display = 'none';
            localVideo.style.display = 'block'; // Show local video overlay
        }


        // --- 3. PEERJS & CONNECTION HANDLERS ---

        function initializePeer() {
            // Initialize PeerJS
            peer = new Peer(null, {
                host: 'peerjs.com',
                port: 443,
                secure: true
            });

            peer.on('open', (id) => {
                peerId = id;
                setStatus(`Your ID: ${peerId.substring(0, 8)}... (Ready)`, 'var(--accent-green)');
                if (!dataConn) { // Only enable input for the first time
                    chatInput.disabled = false;
                    sendIcon.style.color = 'var(--accent-green)';
                }
            });

            peer.on('error', (err) => {
                console.error('PeerJS Error:', err);
                setStatus(`Connection Error: ${err.type}`, 'red');
            });
            
            // Incoming Data/Chat Connection
            peer.on('connection', (newConn) => {
                handleDataConnection(newConn);
                setStatus(`Incoming Chat from: ${newConn.peer.substring(0, 8)}...`, 'yellow');
                showChatView(); 
            });

            // Incoming Media/Video Call
            peer.on('call', (call) => {
                setStatus('Incoming Video Call... Answering.', 'orange');
                startLocalMedia()
                    .then(stream => {
                        call.answer(stream);
                        handleMediaCall(call);
                        showVideoView();
                    })
                    .catch(err => {
                        console.error('Failed to answer call:', err);
                        setStatus('Permission Error: Cannot answer call.', 'red');
                    });
            });
        }

        function handleDataConnection(conn) {
            dataConn = conn;
            chatInput.disabled = false;
            sendIcon.style.color = 'var(--accent-green)';

            dataConn.on('data', (data) => {
                if (data.type === 'chat') {
                    appendMessage(data.message, 'msg-remote');
                }
            });

            dataConn.on('close', () => {
                setStatus('Chat Connection closed.', 'red');
                chatInput.disabled = true;
                sendIcon.style.color = 'var(--icon-color)';
                dataConn = null;
            });
        }

        function handleMediaCall(call) {
            mediaCall = call;
            call.on('stream', (remoteStream) => {
                remoteVideo.srcObject = remoteStream;
                setStatus('Video Call Active!', 'var(--accent-green)');
            });
            call.on('close', () => {
                setStatus('Call Ended.', 'red');
                mediaCall = null;
                remoteVideo.srcObject = null;
                showChatView();
            });
        }

        function startLocalMedia() {
            // Request camera and microphone access
            return navigator.mediaDevices.getUserMedia({ video: true, audio: true })
                .then(stream => {
                    localStream = stream;
                    localVideo.srcObject = stream;
                    return stream;
                });
        }

        function connectToPeer(remotePeerId) {
            if (!peerId) { setStatus('Peer not ready. Please wait.', 'red'); return; }
            if (dataConn) { setStatus('Already connected.', 'yellow'); return; }

            setStatus(`Connecting chat to ${remotePeerId.substring(0, 8)}...`, 'yellow');

            // 1. Establish Data Connection
            const conn = peer.connect(remotePeerId);
            conn.on('open', () => {
                handleDataConnection(conn);
                setStatus('Chat Connection Established! Ready to send messages.', 'var(--accent-green)');
                showChatView();
            });
        }
        
        function initiateVideoCall(remotePeerId) {
            if (!peerId) { setStatus('Peer not ready.', 'red'); return; }
            if (mediaCall) { setStatus('Video call already active.', 'yellow'); return; }
            
            setStatus(`Starting video call to ${remotePeerId.substring(0, 8)}...`, 'orange');

            // Start media first
            startLocalMedia()
                .then(stream => {
                    const call = peer.call(remotePeerId, stream);
                    handleMediaCall(call);
                    showVideoView();
                })
                .catch(err => {
                    console.error('Failed to start media call:', err);
                    setStatus('Error: Cannot start call without camera/mic.', 'red');
                });
        }

        function endCall() {
            if (mediaCall) {
                mediaCall.close();
            }
            if (localStream) {
                localStream.getTracks().forEach(track => track.stop());
                localStream = null;
            }
            setStatus('Call/Connection ended.', 'red');
            showChatView();
        }

        // --- 4. QR CODE & SCANNING LOGIC ---
        
        function generateQRCode(text) {
            qrCanvas.style.display = 'block';
            scannerVideo.style.display = 'none';
            document.getElementById('qr-title').textContent = 'Your Peer ID';
            document.getElementById('qr-instructions').textContent = `Scan this QR to connect to your ID: ${peerId.substring(0, 8)}...`;
            QRCode.toCanvas(qrCanvas, text, function (error) {
                if (error) console.error(error);
            });
        }

        function startQRScanner() {
            qrCanvas.style.display = 'none';
            scannerVideo.style.display = 'block';
            document.getElementById('qr-title').textContent = 'Scanning...';
            document.getElementById('qr-instructions').textContent = 'Point your camera at the other user\'s QR ID.';

            Instascan.Camera.getCameras().then(function (cameras) {
                if (cameras.length > 0) {
                    // Try to use the back camera first, if available (assumes mobile)
                    const backCamera = cameras.find(c => c.name.toLowerCase().includes('back')) || cameras[0];
                    scanner.start(backCamera);
                } else {
                    document.getElementById('qr-instructions').textContent = 'Error: No camera detected for scanning.';
                }
            }).catch(function (e) {
                console.error(e);
                document.getElementById('qr-instructions').textContent = 'Error accessing camera: Check permissions.';
            });

            scanner.addListener('scan', function (content) {
                scanner.stop();
                // Basic validation for Peer ID format
                if (content && content.length > 10) { 
                    connectToPeer(content);
                } else {
                    alert('Invalid Peer ID format received.');
                }
            });
        }

        // --- 5. CHAT MESSAGE LOGIC ---

        function appendMessage(message, type) {
            // Remove initial placeholder text
            const initialMsg = document.getElementById('initial-message');
            if (initialMsg) initialMsg.remove();
            
            const messageElement = document.createElement('div');
            messageElement.className = `message-bubble ${type}`;
            messageElement.textContent = message;
            messageDisplay.appendChild(messageElement);
            // Auto-scroll to the bottom
            messageDisplay.scrollTop = messageDisplay.scrollHeight;
        }

        function sendMessage() {
            const message = chatInput.value.trim();
            if (message && dataConn && dataConn.open) {
                // Send an object containing the message type
                dataConn.send({ type: 'chat', message: message });
                appendMessage(message, 'msg-local');
                chatInput.value = '';
            } else if (message) {
                 setStatus('Not connected to a peer for chat.', 'red');
                 chatInput.value = '';
            }
        }


        // --- 6. INITIALIZATION & EVENT LISTENERS ---
        document.addEventListener('DOMContentLoaded', () => {
            initializePeer();
            
            // Tab Navigation Listeners
            document.getElementById('chat-tab').addEventListener('click', showChatView);
            document.getElementById('scan-qr-tab').addEventListener('click', () => showQRView('scan'));
            document.getElementById('show-id-tab').addEventListener('click', () => showQRView('show'));
            document.getElementById('cancel-qr-button').addEventListener('click', showChatView);

            // QR Actions Listeners (for explicit buttons)
            document.getElementById('scan-button').addEventListener('click', () => startQRScanner());
            document.getElementById('show-my-qr-button').addEventListener('click', () => generateQRCode(peerId));
            
            // Message Sending Listeners
            sendIcon.addEventListener('click', sendMessage);
            chatInput.addEventListener('keypress', (e) => {
                if (e.key === 'Enter') sendMessage();
            });

            // Video Call Controls
            document.getElementById('start-video-call').addEventListener('click', () => {
                if (dataConn) {
                    // Use the connected peer's ID to start the media call
                    initiateVideoCall(dataConn.peer); 
                } else {
                    setStatus('Start chat connection first using QR code.', 'yellow');
                }
            });
            document.getElementById('end-call-button').addEventListener('click', endCall);
            
            // Picture-in-Picture Logic
            document.getElementById('pip-button').addEventListener('click', () => {
                if (mediaCall && remoteVideo.srcObject) {
                    if (document.pictureInPictureElement) {
                        document.exitPictureInPicture();
                    } else {
                        remoteVideo.requestPictureInPicture().catch(e => console.error("PiP failed:", e));
                    }
                } else {
                    alert('No active video stream to put into PiP.');
                }
            });
        });
    </script>
</body>
</html>
