  I started from the web UI and RTSP surface on 192.168.1.41 and
  confirmed the camera is XM/Xiongmai-style:

  - HTTP on 80
  - RTSP on 554
  - DVRIP control on 34567

  Initial stream state I verified from live RTP payloads:

  - stream=0: H.265
  - stream=1: H.265
  - stream=3: advertised H.264 in SDP, but did not emit a working stream

  How I accessed configuration:

  - I first looked at the web UI JS and found it only uses /cgi-bin/
    login.cgi for pre-login info, then hands off to a local plugin on
    127.0.0.1:54455-54465 via /Cmd-WebLocalCtrl.
  - I reversed enough of that to conclude the plugin is just a shim to
    the camera’s DVRIP protocol on port 34567.
  - I then bypassed the plugin entirely and spoke DVRIP directly.

  Working DVRIP access:

  - Host: 192.168.1.41:34567
  - Login method: LoginType: DVRIP-Web
  - Username: admin
  - Password: blank
  - Password encoding: XM “sofia hash” used by python-dvr

  I used the same login model as the python-dvr implementation:

  - https://raw.githubusercontent.com/OpenIPC/python-dvr/master/dvrip.py

  Important config objects I found:

  - Simplify.Encode
  - AVEnc
  - EncodeCapability
  - SystemFunction

  What mattered in practice:

  - Simplify.Encode showed main/sub format settings, but changing it was
    not enough
  - AVEnc was the real active runtime encoder object

  Relevant live config findings:

  - Simplify.Encode runtime (1042) had substream as H.265
  - Simplify.Encode default config (1044) had substream as H.264
  - AVEnc contained the active formats:
      - AVEnc.Encode[0].MainFormat[0].Video.Compression = H.265
      - AVEnc.Encode[0].ExtraFormat[*].Video.Compression = H.265

  Critical change:

  - I changed all AVEnc.Encode[0].ExtraFormat[*].Video.Compression
    entries from H.265 to H.264
  - The write succeeded and read back correctly through DVRIP
  - The running encoder did not switch immediately
  - After reboot, the change took effect

  After reboot:

  - AVEnc extra formats were still H.264
  - RTSP stream=1 changed to H.264
  - I verified from live RTP payloads, not just SDP:
      - SPS 0x67
      - PPS 0x68
      - FU-A 0x7c
      - slice 0x21

  Final state at handoff:

  - stream=0: still H.265
  - stream=1: working H.264
  - stream=3: still not useful; it advertises H.264 but was not needed
    once stream=1 became real H.264

  Working H.264 RTSP URL:

  - rtsp://192.168.1.41/user=admin_password=_channel=1_stream=1.sdp

  If Codex on the other host wants the exact practical approach:

  1. Login to DVRIP on 34567
  2. Read AVEnc
  3. Modify AVEnc.Encode[0].ExtraFormat[*].Video.Compression to H.264
  4. Write AVEnc back
  5. Reboot camera
  6. Verify stream=1 via live RTP payload, not only SDP

  Main lesson:

  - Simplify.Encode is not enough
  - AVEnc is the effective configuration path
  - reboot was required for the media pipeline to rebind to the new
    codec

