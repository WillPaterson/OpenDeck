<script lang="ts">
	import type { DeviceInfo } from "$lib/DeviceInfo";
	import type { Profile } from "$lib/Profile";

	import { settings } from "$lib/settings";
	import { profileManager } from "$lib/singletons";

	import { invoke } from "@tauri-apps/api/core";
	import { listen } from "@tauri-apps/api/event";
	import { getCurrentWindow, LogicalSize } from "@tauri-apps/api/window";
	import Star from "phosphor-svelte/lib/Star";
	import X from "phosphor-svelte/lib/X";
	import Tooltip from "./Tooltip.svelte";

	export let devices: { [id: string]: DeviceInfo } = {};
	export let value: string;
	export let selectedProfiles: { [id: string]: Profile } = {};

	let registered: string[] = [];
	$: {
		if (!value || !devices[value]) value = Object.keys(devices).sort()[0];
		for (const [id, device] of Object.entries(devices)) {
			if (!registered.includes(id)) {
				(async () => {
					let profile: Profile = await invoke("get_selected_profile", { device: device.id });
					selectedProfiles[id] = profile;
					await invoke("set_selected_profile", { device: id, id: profile.id });
				})();
				registered.push(id);
			}
		}
	}

	export function reloadProfiles() {
		registered = [];
	}

	listen("switch_profile", async ({ payload }: { payload: { device: string; profile: string } }) => {
		if (payload.device == value) {
			$profileManager?.setProfile(payload.profile);
		} else {
			await invoke("set_selected_profile", { device: payload.device, id: payload.profile });
			selectedProfiles[payload.device] = await invoke("get_selected_profile", { device: payload.device });
		}
	});

	(async () => devices = await invoke("get_devices"))();
	listen("devices", ({ payload }: { payload: { [id: string]: DeviceInfo } }) => devices = payload);

	let buildInfo: string;
	(async () => buildInfo = await invoke("get_build_info"))();
	const window = getCurrentWindow();

	$: {
		if (devices[value]) {
			const effectiveCols = Math.min(Math.max(devices[value].columns, devices[value].encoders, devices[value].touchpoints), 8);
			const effectiveRows = Math.min(devices[value].rows + Math.min(devices[value].encoders, 1) + Math.min(devices[value].touchpoints, 1), 4);
			const idealWidth = (effectiveCols * 132) + 416;
			const idealHeight = (effectiveRows * 132) + 384 + (buildInfo?.split("</summary>")[0]?.includes("darwin") ? 28 : 0);
			(async () => {
				const width = Math.min(idealWidth, screen.availWidth);
				const height = Math.min(idealHeight, screen.availHeight);
				await window.setMinSize(new LogicalSize(width, height));
				await window.setSize(new LogicalSize(width, height));
			})();
		}
	}

	let measure: HTMLSpanElement;
	let selectWidth = 0;
	$: if (value && measure && devices[value]) {
		measure.textContent = deviceName(devices[value]);
		selectWidth = measure.offsetWidth + 20;
	}

	function deviceName(device: DeviceInfo) {
		return device.connected ? device.name : `${device.name} (Disconnected)`;
	}

	function setDefaultDevice() {
		if (!$settings || !devices[value]) return;
		const device = devices[value];
		$settings = {
			...$settings,
			default_device: {
				id: device.id,
				name: device.name,
				rows: device.rows,
				columns: device.columns,
				encoders: device.encoders,
				touchpoints: device.touchpoints,
				type: device.type,
			},
		};
	}

	function clearDefaultDevice() {
		if (!$settings) return;
		const wasOfflineDefault = value && devices[value] && !devices[value].connected && $settings.default_device?.id == value;
		$settings = { ...$settings, default_device: null };
		if (wasOfflineDefault) {
			delete devices[value];
			devices = devices;
			value = Object.keys(devices).sort()[0] ?? "";
		}
	}
</script>

{#if Object.keys(devices).length > 0}
	<div class="flex flex-row items-center space-x-2">
		<div class="select-device-wrapper">
			<span bind:this={measure} class="invisible fixed whitespace-pre pointer-events-none text-xl font-semibold" aria-hidden="true"></span>
			<select bind:value style:width="{selectWidth}px" aria-label="Device">
				<option value="" disabled selected>Choose a device...</option>

				{#each Object.entries(devices).sort() as [id, device]}
					<option value={id}>{deviceName(device)}</option>
				{/each}
			</select>
		</div>

		{#if $settings}
			{#if $settings.default_device?.id == value}
				<button
					class="flex items-center gap-2 p-1 text-yellow-400 hover:text-neutral-300 transition-colors"
					on:click={clearDefaultDevice}
					title="Clear the default device so new devices won't duplicate this one?"
					aria-label="Clear the default device so new devices won't duplicate this one?"
				>
					<X size={18} />
					Clear default?
				</button>
			{:else}
				<button
					class="flex items-center gap-2 p-1 text-neutral-500 hover:text-yellow-400 transition-colors"
					on:click={setDefaultDevice}
					title="Set this device as the default source to duplicate when connecting new devices?"
					aria-label="Set this device as the default source to duplicate when connecting new devices?"
				>
					<Star size={18} />
					Set as default?
				</button>
			{/if}

			<Tooltip>
				{#if $settings.default_device}
					Current Default: {deviceName(devices[$settings.default_device.id] ?? { ...$settings.default_device, connected: false })}
				{/if}

				<br />

				Setting a default device will cause new devices to duplicate its configuration when they connect for the first time.
			</Tooltip>
		{/if}
	</div>
{/if}
