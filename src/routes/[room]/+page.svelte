<script lang="ts">
	import { onMount } from "svelte";
    import { page } from "$app/stores"
	import {
		relayInit,
		generatePrivateKey,
		getPublicKey,
		getEventHash,
		signEvent,
		nip04,
		type Event,
	} from "nostr-tools";

	const RTC_CONFIG = { iceServers: [{ urls: "stun:stun.l.google.com:19302" }] };
	const relay = relayInit("wss://nostr.fmt.wiz.biz");
	console.log("hello");
	const peers: any = {};

	let remoteStream: any;
	let remoteAudioElement;

	onMount(() => {
        remoteStream = new MediaStream();
		remoteAudioElement = document.querySelector("#remoteAudio");
		// @ts-ignore
		remoteAudioElement!.srcObject = remoteStream;

        navigator.mediaDevices
		.getUserMedia({ audio: true })
		.then((stream) => {
			localStream = stream;

			// analyse stream to get volume of input sound
			getSoundVolume(localStream.clone(), (vol: number) => {
				// set volume
				volume = vol

				// permit audio transmission when
				// vol is higher then threshold
				localStream.getAudioTracks()[0].enabled = vol > threshold;
			});

			relay.connect();
			mediaDeviceInit = true
		})
		.catch((err) => {
			mediaDeviceInit = false
		});
	});

	let localStream: any;
	let threshold: any = 10;
	let volume: any = 0;
	let mediaDeviceInit: any = null;
	let users: any = [];
	let privkey = generatePrivateKey();
	let pubkey = getPublicKey(privkey);
	console.log(pubkey);
    let room = $page.params.room

	async function postToRelay(type: string, mPubkey: string, msg: any) {
		let event: Event = {
			kind: 25050,
			id: "",
			sig: "",
			pubkey,
			created_at: Math.floor(Date.now() / 1000),
			tags: [
				["type", type],
			],
			content: ""
		};
		switch (type) {
			case "answer":
			case "offer":
			case "candidate":
				event.tags.push(["p", mPubkey])
				event.tags.push(["r", await nip04.encrypt(privkey, mPubkey, room)])
				event.content = await nip04.encrypt(privkey, mPubkey, JSON.stringify(msg))
				break;
			default:
				event.tags.push(["r", room])
		}
		event.id = getEventHash(event);
		event.sig = signEvent(event, privkey);
		let pub = relay.publish(event);
		pub.on("ok", () => {
			console.log(`${relay.url} has accepted our event`);
		});
	}

	relay.on("connect", () => {
		console.log("connected");
		postToRelay("connect", "", "");
		let sub = relay.sub([
			{
				kinds: [25050],
				"#r": [room],
			},
			{
				kinds: [25050],
				"#p": [pubkey],
			},
		]);
		sub.on("event", async (event: Event) => {
			if (event.pubkey === pubkey) return;
			console.log("found event", event);
			let newRoom: string = ""
			let content: string = ""
			if (event.tags.find((v) => v[0] == "p")) {
				newRoom = await nip04.decrypt(privkey, event.pubkey, event.tags.find((v) => v[0] == "r")![1])
				if (newRoom !== room) return;
				content = await nip04.decrypt(privkey, event.pubkey, event.content)
			}
			let type = event.tags.find((v) => v[0] == "type")
			if (!type) return
			switch (type[1]) {
				case "connect":
					initRTCPeerConnection(event.pubkey);
					break;
				case "disconnect":
					removeRTCPeerConnection(event.pubkey);
					break;
				case "answer":
					onRTCAnswer(event.pubkey, JSON.parse(content));
					break;
				case "offer":
					onRTCoffer(event.pubkey, JSON.parse(content));
					break;
				case "candidate":
					onRTCIceCandidate(event.pubkey, JSON.parse(content));
					break;
				default:
					break;
			}
		});
	});

	const onRTCIceCandidate = async (id: string, candidate: any) => {
		console.log(`got ice candidate from ${id}`, candidate);

		if (!candidate) return;

		const pc = peers[id];

		if (!pc) return;

		await pc.addIceCandidate(candidate);
	};

	const removeRTCPeerConnection = (id: string) => {
		const pc = peers[id];

		if (!pc) return;

		pc.close();

		delete peers[id];

		users = Object.keys(peers)

		console.log(`removed rtc peer connection ${id}`);
	};

	const initRTCPeerConnection = async (id: string) => {
		const pc = new RTCPeerConnection(RTC_CONFIG);

		addLocalStream(pc);
		addRemoteStream(pc);

		pc.onicecandidate = sendIceCandidate(id);

		// add peerconnection to peerlist
		peers[id] = pc;

		// update userlist
		users = Object.keys(peers)

		// create a new offer
		const offer = await pc.createOffer();

		// set offer as local descrioption
		await pc.setLocalDescription(offer);

		// send offer out
		postToRelay("offer", id, offer);

		// log
		console.log(`init new rtc peer connection for client ${id}`, offer);
	};

	const onRTCAnswer = async (id: string, answer: any) => {
		console.log(`got answer from ${id}`, answer);

		const pc = peers[id];

		if (!pc) return;

		if (!answer) return;

		const desc = new RTCSessionDescription(answer);

		await pc.setRemoteDescription(desc);
	};

	const onRTCoffer = async (id: string, offer: any) => {
		console.log(`got offer from ${id}`, offer);

		if (!offer) return;

		const pc = new RTCPeerConnection(RTC_CONFIG);

		addLocalStream(pc);
		addRemoteStream(pc);

		pc.onicecandidate = sendIceCandidate(id);

		peers[id] = pc;

		users = Object.keys(peers)

		const desc = new RTCSessionDescription(offer);

		pc.setRemoteDescription(desc);

		const answer = await pc.createAnswer();

		await pc.setLocalDescription(answer);

		postToRelay("answer", id, answer);
	};

	const sendIceCandidate =
		(id: string) =>
		({ candidate }: any) => {
			if (candidate) {
				postToRelay("candidate", id, candidate);
			}
		};

	const addLocalStream = (pc: any) => {
		localStream.getTracks().forEach((track: any) => pc.addTrack(track, localStream));
	};

	const addRemoteStream = (pc: any) => {
		pc.ontrack = async (evt: any) => {
			remoteStream.addTrack(evt.track);
		};
	};

	const getSoundVolume = (stream: any, callback: any) => {
		const audioCtx = new AudioContext();
		const analyser = audioCtx.createAnalyser();
		const mic = audioCtx.createMediaStreamSource(stream);
		const node = audioCtx.createScriptProcessor(2048, 1, 1);

		// analyser args
		analyser.smoothingTimeConstant = 0.8;
		analyser.fftSize = 1024;

		mic.connect(analyser);
		analyser.connect(node);
		node.connect(audioCtx.destination);

		node.onaudioprocess = () => {
			let array = new Uint8Array(analyser.frequencyBinCount);
			analyser.getByteFrequencyData(array);

			let values = 0;

			for (let i = 0; i < array.length; i++) {
				values += array[i];
			}

			const avg = values / array.length;

			callback(avg);
		};
	};
</script>

<div class="app">
	<div class="panel">
        YOU ARE: {pubkey}
		<h4>Userlist</h4>
		<ul>
			{#each users as uid, i}
				<li>{uid}</li>
			{/each}
		</ul>
	</div>
	<div class="panel">
		<h4>Settings</h4>
		<p>
			<input type="range" min="0" max="100" class="slider" bind:value={threshold} />
            <br />
            THRESHOLD SLIDER - ALL THE WAY DOWN TO MAKE YOUR MICROPHONE MORE SENSITIVE AND VICE-VERSA
		</p>
		<p>
			<progress class="progress" class:green={volume >= threshold} value={volume} max="100" />
            <br />
            MIC ACTIVITY
		</p>
	</div>
</div>


