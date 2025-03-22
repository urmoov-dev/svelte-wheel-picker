<script module lang="ts">
// Export Types

export type Options = {
	visibleOptions?: number
	density?: number
	marginY?: number
	perspective?: number
	staticPointerTimer?: number
	friction?: number
	overflow?: "hidden" | "visible"
}
export type FillParentOn = {
	disable: false
	height?: undefined
	width?: undefined
}
export type FillParentOff = {
	disable: true
	height?: string
	width?: string
}
export type HoverEffectOff = {
	enable: false
	unwrap?: undefined
	stamp?: undefined
	fixList?: undefined
	transformSpeed?: undefined
}
export type HoverEffectOn = {
	enable: true
	unwrap?: number
	stamp?: boolean
	fixList?: boolean
	transformSpeed?: number
}
export type DataOption = { value: number | string; label: string }
</script>

<script lang="ts">
import { run, preventDefault, stopPropagation, nonpassive } from "svelte/legacy"

interface Props {
	//Props
	classes?: string
	style?: string
	data?: DataOption[]
	selectedOption?: DataOption
	rollOnHover?: HoverEffectOff | HoverEffectOn
	fillParent?: FillParentOff | FillParentOn
	options?: Options
}

let {
	classes = "",
	style = "",
	data = [
		{ value: 5, label: "This is" },
		{ value: 10, label: "only a" },
		{ value: 15, label: "placeholder " },
		{ value: 20, label: "you should " },
		{ value: 25, label: "send your" },
		{ value: 30, label: "data" },
		{ value: 35, label: "in the" },
		{ value: 40, label: "'wheelOptions'" },
		{ value: 45, label: "prop and" },
		{ value: 50, label: "configure" },
		{ value: 55, label: "your" },
		{ value: 60, label: "settings" },
		{ value: 70, label: "with the" },
		{ value: 80, label: "following" },
		{ value: 90, label: "props:" },
		{ value: 100, label: "marginY:number" },
		{ value: 120, label: "fillParent:boolean" },
		{ value: 130, label: "visibleOptions:number" },
		{ value: 145, label: "perspective:number" },
		{ value: 160, label: "bearingFriction:number" },
	],
	selectedOption = $bindable(data[0]),
	rollOnHover = { enable: false },
	fillParent = { disable: false },
	options = {},
}: Props = $props()

// Local State

// not using destructuring to enforce default values if !hoverEnabled
let hoverEnabled = rollOnHover?.enable || false,
	unwrap = $state(!hoverEnabled ? false : (rollOnHover?.unwrap ?? 85)),
	stamp = !hoverEnabled ? false : (rollOnHover?.stamp ?? false),
	fixList = !hoverEnabled ? false : (rollOnHover?.fixList ?? !stamp),
	transformSpeed = !hoverEnabled ? false : (rollOnHover?.transformSpeed ?? 0.3)
let { disable: fillParentDisabled = false, height, width } = fillParent
let {
	visibleOptions = 7,
	density = 14,
	marginY = 2,
	perspective = 0,
	staticPointerTimer = 200,
	friction = 4,
	overflow = hoverEnabled ? "visible" : "hidden",
} = $state(options)
let leaningAngle: number[] = []
let wheelStyle: string[] = $state([])
let yOffset: number = $state(0)
let mousedown: boolean = $state(false)
let touchstart: boolean = $state(false)
let touchstartPosition: number = $state(1)
let dragPosition: number = $state(0)
let resetYInertia: Timer
let prevDisplacement: number = $state(0)
let prevOffset: number = 0
let yInertia: number = 0
let hovering: boolean = $state(!hoverEnabled)
let optionToSelect: { angle: number; index: number } = $state({
	angle: 1,
	index: 0,
})

// Bindings

let wheelWindowWidth: number | undefined = $state()
let wheelWindowHeight: number | undefined = $state()
let wheelContainerHeight: number | undefined = $state()
let wheelWrapperHeight: number | undefined = $state()

// Functions

function grabWheel(e: TouchEvent | MouseEvent) {
	resetDrag()
	if (e.type === "mousedown") {
		mousedown = true
		touchstartPosition = (e as MouseEvent).clientY
	} else if (e.type === "touchstart") {
		touchstart = true
		touchstartPosition = (e as TouchEvent).changedTouches[0]?.clientY ?? 0
	}
}
function releaseWheel(_e: TouchEvent | MouseEvent) {
	mousedown = false
	touchstart = false
	yInertia = yOffset - prevOffset
	continueWheelMomentum()
}
function dragWheel(e: TouchEvent | MouseEvent) {
	if (!mousedown && !touchstart) return

	prevOffset = yOffset

	if (mousedown) {
		dragPosition = (e as MouseEvent).clientY
	} else if (touchstart) {
		dragPosition = (e as TouchEvent).changedTouches[0]?.clientY ?? 0
	}

	resetYInertia = setTimeout(() => {
		prevOffset = yOffset
	}, staticPointerTimer)
}
function generateTransforms(displacement: number, wrap: boolean | number) {
	for (let index = 0; index < data.length; index++) {
		const currentLeaningAngle = (index *
			((180 + perspective) / visibleOptions) -
			displacement) as number
		leaningAngle[index] = currentLeaningAngle
		const currentAngle = wrap
			? currentLeaningAngle
			: currentLeaningAngle - currentLeaningAngle * ((unwrap as number) / 100)

		if (Math.abs(currentAngle) < Math.abs(optionToSelect.angle)) {
			optionToSelect.index = index
		}
		if (index === optionToSelect.index) {
			optionToSelect.angle = currentAngle
		}

		const rotateX = Math.max(Math.min(currentAngle, 90), -90)

		// y = a(x-h)**2 + k where k = 100, y = 0 and x = 0
		// const k = 100
		// const h = Math.max(30)
		// const a = -100 / h ** 2
		// const x = currentAngle
		// const parabola = a * (x - h) ** 2 + k

		// const translateY = !roll
		//         ? 0 : currentAngle >= 0
		//         ? a*(x-h)**2+k
		//         : -a*(x+h)**2-k
		const translateY = wrap ? (rotateX * visibleOptions) / Math.PI : 0

		const absRotation = Math.abs(wrap ? rotateX : currentLeaningAngle)

		const scale = wrap
			? (density - visibleOptions) / density +
				((180 - absRotation) / 180) ** 1.4
			: 1

		const opacity = 0.6 - absRotation / (wrap ? 150 : 300)

		wheelStyle[index] =
			`transform: rotateX(${rotateX}deg) translateY(${translateY}%) scale(${scale}); opacity: ${opacity};`
	}
}
async function asyncTimeout(ms: number) {
	return new Promise((res) => setTimeout(res, ms))
}
async function continueWheelMomentum() {
	if (mousedown || touchstart) return
	yOffset += yInertia
	yInertia *= (100 - friction) / 100
	let absYInertia = Math.abs(yInertia)
	if (
		dragPosition &&
		absYInertia > 0.1 - friction / 100 &&
		pickerDisplacement < 0 &&
		pickerDisplacement > maxScroll
	) {
		await asyncTimeout(8)
		continueWheelMomentum()
	} else {
		centerSelectedOption()
	}
}
function resetDrag() {
	touchstartPosition = 0
	prevDisplacement = Math.max(
		Math.min(prevDisplacement + yOffset, 0),
		maxScroll,
	)
	dragPosition = 0
	prevOffset = 0
}
async function centerSelectedOption() {
	if (mousedown) return
	yOffset -= (optionToSelect.angle * friction) / 100
	if (
		Math.abs(optionToSelect.angle) > 0.5 &&
		dragPosition &&
		pickerDisplacement > maxScroll
	) {
		await asyncTimeout(8)
		centerSelectedOption()
	} else {
		resetDrag()
	}
}

//Reactive declarations

let longestLabel = $derived(
	data.slice().sort((a, b) => (a.label.length > b.label.length ? -1 : 1))[0]!
		.label,
)
//clamp the perspective option to avoid weird rendering
$effect(() => {
	perspective = Math.max(-90, Math.min(stamp ? 180 : 90, perspective))
})
//clamp visible options to reasonable values
$effect(() => {
	visibleOptions = Math.max(3, Math.min(stamp ? 180 : 90, visibleOptions))
})
let maxAngle = $derived(
	((180 + perspective) / visibleOptions) * (data.length - 1),
)
let wrap = $derived(!hoverEnabled || dragPosition || hovering ? true : false)
$effect(() => {
	density = !fillParentDisabled ? visibleOptions * 2 : density
})
let perspectiveScaleOffset = $derived(
	stamp && wrap
		? 1 / (1 + (density - visibleOptions) / density)
		: hoverEnabled
			? 1
			: Math.max(1, 1 + perspective / 200),
)
let fillWidthLineHeight = $derived(
	(wheelWindowWidth * 2) /
		(longestLabel.length * Math.max(1, perspectiveScaleOffset)),
)
let fillHeightLineHeight = $derived(
	Math.round(
		wheelWindowHeight /
			(stamp ? data.length : Math.min(visibleOptions, data.length)),
	) * 0.8,
)
let lineHeight = $derived(Math.min(fillWidthLineHeight, fillHeightLineHeight))
let maxScroll = $derived(-wheelContainerHeight + lineHeight)
$effect(() => {
	yOffset = dragPosition === 0 ? 0 : dragPosition - touchstartPosition
})
let pickerDisplacement = $derived(
	Math.max(Math.min(prevDisplacement + yOffset, 0), maxScroll),
)
let scrollPercent = $derived(pickerDisplacement / maxScroll)
let rotationDisplacement = $derived(maxAngle * scrollPercent)
let fillParentStyle = $derived(
	!fillParentDisabled
		? `height: calc(100%); width: ${longestLabel.length * 2}ch`
		: `${height ? `height: ${height}` : ""}; ${width ? `width: ${width}` : ""};`,
)
let wheelWrapperScale = $derived(`transform: scale(${perspectiveScaleOffset})`)
let wheelContainerTop = $derived(
	!hoverEnabled || (fixList && wrap) || (!fixList && !stamp)
		? `calc(50% - ${lineHeight / 2.15}px)`
		: `0%`,
)
let wheelContainerOffset = $derived(
	hoverEnabled && stamp && wrap
		? -pickerDisplacement * 0.5
		: !hoverEnabled || (!fixList && !stamp) || (wrap && fixList)
			? pickerDisplacement
			: 0,
)
let wheelContainerTransform = $derived(
	`translateY(${wheelContainerTop}) translateY(${wheelContainerOffset}px);`,
)
let fontSize = $derived(lineHeight * 0.8)
$inspect(
	"fontSize",
	fontSize,
	"lineHeight",
	lineHeight,
	"fillWidthLineHeight",
	fillWidthLineHeight,
	"fillHeightLineHeight",
	fillHeightLineHeight,
	"wheelWindowWidth",
	wheelWindowWidth,
	"longestLabel",
	longestLabel,
)
// Reactive statements

run(() => {
	generateTransforms(rotationDisplacement, wrap)
})
run(() => {
	if (fillParentDisabled && !height) {
		console.warn(
			"The fillParent prop is disabled but its `height` property is not set, the WheelPicker will automatically fill the parent until the height is set.",
		)
	}
})
run(() => {
	if ((unwrap as number) > 99) {
		unwrap = 99
		console.warn(
			"The `unwrap` property in the `rollOnHover` prop cannot be higher than 99",
		)
	}
})
run(() => {
	if (data[optionToSelect.index])
		selectedOption = data[optionToSelect.index] as DataOption
})
</script>

<svelte:window
	onmouseup={stopPropagation(
		preventDefault(releaseWheel as (e: Event) => void),
	)}
	ontouchend={stopPropagation(
		preventDefault(releaseWheel as (e: Event) => void),
	)}
	onmousemove={stopPropagation(preventDefault(dragWheel as (e: Event) => void))}
	use:nonpassive={[
		"touchmove",
		() => stopPropagation(preventDefault(dragWheel as (e: Event) => void)),
	]}
/>

<div
	class="wheelWindow {classes}"
	role="button"
	tabindex="0"
	style="{fillParentStyle};
                padding: {marginY}% 0px;
                font-size: {fontSize}px;
                line-height:{lineHeight}px;
                overflow: {overflow};
                {style};
            "
	onmousedown={stopPropagation(preventDefault(grabWheel as (e: Event) => void))}
	use:nonpassive={[
		"touchstart",
		() => stopPropagation(preventDefault(grabWheel as (e: Event) => void)),
	]}
	onmouseenter={() => {
		if (rollOnHover.enable) hovering = true
	}}
	onmouseleave={() => {
		if (rollOnHover.enable && !mousedown && !touchstart) hovering = false
	}}
	bind:clientHeight={wheelWindowHeight}
	bind:clientWidth={wheelWindowWidth}
>
	<div
		class="wheelWrapper"
		style="
                {wheelWrapperScale};
                {hoverEnabled && stamp
			? `transition: transform ${transformSpeed}s ease-in-out`
			: ''};
                transform-origin: {hoverEnabled && stamp ? 'top' : 'center'};
            "
		bind:clientHeight={wheelWrapperHeight}
	>
		<div
			class="wheelContainer"
			bind:clientHeight={wheelContainerHeight}
			style=" transform: {wheelContainerTransform};
                        height: {Math.max(
				wheelWrapperHeight,
				lineHeight * data.length,
			)}px;
                        {!dragPosition
				? `transition: transform ${transformSpeed}s ease-in-out`
				: ''};
                      "
		>
			{#each data as option, i (i)}
				<span
					class="wheelOption"
					style=" {wheelStyle[i]};
                                {i === optionToSelect.index
						? 'opacity: 1'
						: ''};
                                {(hoverEnabled && !dragPosition) ||
					!dragPosition
						? `transition: transform ${transformSpeed}s ease-in-out, opacity ${transformSpeed}s ease-in-out`
						: ''}
                              "
				>
					{option.label}
				</span>
			{/each}
		</div>
	</div>
</div>

<style>
* {
	box-sizing: border-box;
}

.options {
	display: flex;
	flex-direction: column;
}
.wheelWindow {
	min-height: 4rem;
	height: 100%;
	width: 100%;
	cursor: grab;
	font-family:
		Arial,
		-apple-system,
		BlinkMacSystemFont,
		"Segoe UI",
		Roboto,
		Oxygen,
		Ubuntu,
		Cantarell,
		"Open Sans",
		"Helvetica Neue",
		sans-serif;
}

.wheelWrapper {
	height: 100%;
	width: 100%;
	position: relative;
	display: flex;
	align-items: center;
}

.wheelContainer {
	position: absolute;
	display: flex;
	flex-direction: column;
	justify-content: space-between;
	align-items: center;
	width: 100%;
}

span {
	white-space: nowrap;
	vertical-align: middle;
}
</style>
