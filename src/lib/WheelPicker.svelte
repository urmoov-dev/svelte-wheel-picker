<script context="module" lang="ts">
// Export Types

export type Options = {
        visibleOptions?: number,
        density?: number,
        marginY ?: number,
        perspective?: number,
        staticPointerTimer?: number,
        friction?: number,
        overflow?: "hidden" | "visible"
    };
export type FillParentOn = {
        disable: false
        height?:undefined
        width?:undefined
    };
export type FillParentOff = {
        disable: true,
        height?: string
        width?: string
    };
export type HoverEffectOff = {
        enable: false,
        unwrap?: undefined,
        stamp?: undefined,
        fixList?: undefined,
        transformSpeed?: undefined,
    };
export type HoverEffectOn = {
        enable: true,
        unwrap?: number,
        stamp?: boolean,
        fixList?: boolean,
        transformSpeed?: number,
};
export type DataOption = {value: number|string, label: string}
</script>
<script lang="ts"> 
        
//Props

    export let classes:string = ""
    export let style:string = ""
    export let data:DataOption[] = [
        {value: 5, label: "This is"},
        {value: 10, label: "only a"},
        {value: 15, label: "placeholder "},
        {value: 20, label: "you should "},
        {value: 25, label: "send your"},
        {value: 30, label: "data"},
        {value: 35, label: "in the"},
        {value: 40, label: "'wheelOptions'"},
        {value: 45, label: "prop and"},
        {value: 50, label: "configure"},
        {value: 55, label: "your"},
        {value: 60, label: "settings"},
        {value: 70, label: "with the"},
        {value: 80, label: "following"},
        {value: 90, label: "props:"},
        {value: 100, label: "marginY:number"},
        {value: 120, label: "fillParent:boolean"},
        {value: 130, label: "visibleOptions:number"},
        {value: 145, label: "perspective:number"},
        {value: 160, label: "bearingFriction:number"},
    ];
    export let selectedOption:DataOption = data[0]
    export let rollOnHover:HoverEffectOff|HoverEffectOn = {enable:false};
    export let fillParent:FillParentOff|FillParentOn = {disable:false};
    export let options: Options = {};
    
        
// Local State

    // not using destructuring to enforce default values if !hoverEnabled
    let hoverEnabled = rollOnHover?.enable || false, 
        unwrap = !hoverEnabled ? false : rollOnHover?.unwrap ?? 85, 
        stamp = !hoverEnabled ? false : rollOnHover?.stamp ?? false,
        fixList = !hoverEnabled ? false : rollOnHover?.fixList ?? !stamp, 
        transformSpeed = !hoverEnabled ? false : rollOnHover?.transformSpeed ?? 0.3 
    let {disable: fillParentDisabled = false, height, width} = fillParent;
    let {   visibleOptions = 7,
            density = visibleOptions*2,
            marginY = 2,
            perspective = 0,
            staticPointerTimer = 200,
            friction = 4,
            overflow = hoverEnabled ? "visible" : "hidden"
        } = options;
    let leaningAngle:number[] = [];
    let wheelStyle:string[] = [];
    let yOffset:number = 0;
    let mousedown:boolean = false;
    let touchstart:boolean = false;
    let touchstartPosition:number = 1;
    let dragPosition:number = 0;
    let resetYInertia:number;
    let prevDisplacement:number = 0 ;
    let prevOffset:number = 0;
    let yInertia:number = 0;
    let hovering:boolean = !hoverEnabled;
    let optionToSelect:{angle:number, index: number} = {angle:1, index: 0};

//Reactive declarations

    $: longestLabel = data.slice().sort((a, b) => a.label.length > b.label.length ? -1 : 1)[0].label;
    $: maxAngle = ((180+perspective)/visibleOptions) * (data.length-1);
    $: maxScroll =  -wheelContainerHeight + lineHeight;
    $: pickerDisplacement = Math.max(Math.min(prevDisplacement+yOffset, 0), maxScroll);
    $: yOffset = dragPosition === 0 ? 0 : dragPosition - touchstartPosition;
    $: scrollPercent = pickerDisplacement/maxScroll;
    $: rotationDisplacement = maxAngle*scrollPercent;
    $: fillParentStyle = !fillParentDisabled
            ? `height: calc(100%); width: ${longestLabel.length * 2}ch` 
            : `${height ? `height: ${height}` : ""}; ${width ? `width: ${width}` : ""};`
    $: wrap = !hoverEnabled || (dragPosition || hovering) ? true : false;
    $: perspectiveScaleOffset = stamp && wrap ? 1/(1+(density-visibleOptions)/density) : hoverEnabled ? 1 : Math.max(1, 1+perspective/200)
    $: wheelWrapperScale = `transform: scale(${perspectiveScaleOffset})`;
    $: wheelContainerTop = !hoverEnabled || (fixList && wrap) || (!fixList && !stamp) 
            ? `calc(50% - ${lineHeight/2.15}px)` : `0%` ;
    $: wheelContainerOffset = hoverEnabled && (stamp && wrap) ? -pickerDisplacement*0.5 
            : !hoverEnabled || (!fixList && !stamp) || (wrap && fixList)
            ? pickerDisplacement : 0;
    $: wheelContainerTransform = `translateY(${wheelContainerTop}) translateY(${wheelContainerOffset}px);`;
    $: fillWidthLineHeight = (wheelWindowWidth*2)/(longestLabel.length*Math.max(1, perspectiveScaleOffset));
    $: fillHeightLineHeight = Math.round((wheelWindowHeight) / (stamp ? data.length : Math.min(visibleOptions, data.length)))*0.8;
    $: lineHeight = Math.min(fillWidthLineHeight, fillHeightLineHeight);
    $: fontSize = lineHeight*0.8

// Reactive statements

    $: generateTransforms(rotationDisplacement, wrap);
    //clamp the perspective option to avoid weird rendering
    $: perspective = Math.max(-90, Math.min(stamp ? 180 : 90, perspective)) 
    //clamp visible options to reasonable values
    $: visibleOptions = Math.max(3, Math.min(stamp ? 180 : 90, visibleOptions)) 
    $: density = !fillParentDisabled ? visibleOptions*2 : density
    $: if (fillParentDisabled && !height) {
            console.warn("The fillParent prop is disabled but its `height` property is not set, the WheelPicker will automatically fill the parent until the height is set.");
        };
    $: if (unwrap as number > 99 ) {
            unwrap = 99; 
            console.warn("The `unwrap` property in the `rollOnHover` prop cannot be higher than 99");
        };
    $: selectedOption = data[optionToSelect.index];

// Bindings

    let wheelWindowWidth:number;
    let wheelWindowHeight:number;
    let wheelContainerHeight:number;
    let wheelWrapperHeight:number
    
// Functions

    function grabWheel(e:TouchEvent|MouseEvent) {
        resetDrag()
        if (e.type === "mousedown") {
            mousedown = true;
            touchstartPosition = (e as MouseEvent).clientY;
        }
        else if (e.type === "touchstart") {
            touchstart = true;
            touchstartPosition = (e as TouchEvent).changedTouches[0].clientY;
        };
    }
    function releaseWheel(e:TouchEvent|MouseEvent) {
        mousedown = false;
        touchstart = false;
        yInertia = yOffset - prevOffset;
        continueWheelMomentum();
    }
    function dragWheel(e:TouchEvent|MouseEvent) {
        if (!mousedown && !touchstart) return;
        
        prevOffset = yOffset;
        
        if (mousedown) { 
            dragPosition = (e as MouseEvent).clientY;
        }
        else if (touchstart) {
            dragPosition = (e as TouchEvent).changedTouches[0].clientY;
        };

        resetYInertia = setTimeout(() => {
            prevOffset = yOffset;
        }, staticPointerTimer);
    }
    function generateTransforms(displacement:number, wrap:boolean|number) {
        for (let index = 0; index < data.length; index++) {

            leaningAngle[index] =  (index * ((180+perspective)/visibleOptions)) - displacement;
            const currentAngle = wrap 
                    ? leaningAngle[index] :
                    leaningAngle[index]-(leaningAngle[index]*(unwrap as number/100));
            
            if (Math.abs(currentAngle) < Math.abs(optionToSelect.angle)) {
                optionToSelect.index = index;
            };
            if (index === optionToSelect.index ) {
                optionToSelect.angle = currentAngle;
            };

            const rotateX = Math.max(Math.min(currentAngle, 90), -90);

            // y = a(x-h)**2 + k where k = 100, y = 0 and x = 0
            const k = 100
            const h = Math.max(30)
            const a = -100/h**2
            const x = currentAngle
            const parabola = a*(x-h)**2+k

            // const translateY = !roll 
            //         ? 0 : currentAngle >= 0
            //         ? a*(x-h)**2+k
            //         : -a*(x+h)**2-k
            const translateY = wrap 
                    ? rotateX*visibleOptions/Math.PI 
                    : 0
            
            const absRotation = Math.abs(wrap ? rotateX : leaningAngle[index]);

            const scale = wrap ? (density-visibleOptions)/density+((180-absRotation)/180)**1.4 : 1
            
            const opacity = 0.6-absRotation/(wrap ? 150 : 300);
            
            wheelStyle[index] = `transform: rotateX(${rotateX}deg) translateY(${translateY}%) scale(${scale}); opacity: ${opacity};`;
        }
    };
    async function asyncTimeout(ms:number) {
        return new Promise(res => setTimeout(res, ms));
    };
    async function continueWheelMomentum() {
        if (mousedown || touchstart) return;
        yOffset += yInertia;
        yInertia *= (100-friction)/100;
        let absYInertia = Math.abs(yInertia);
        if (dragPosition && absYInertia > 0.1-friction/100 && pickerDisplacement < 0 && pickerDisplacement > maxScroll) {
            await asyncTimeout(8);
            continueWheelMomentum()
        } else {
            centerSelectedOption()
        };
    };
    function resetDrag() {
        touchstartPosition = 0;
        prevDisplacement = Math.max(Math.min(prevDisplacement+yOffset, 0), maxScroll);
        dragPosition = 0;
        prevOffset = 0;
    };
    async function centerSelectedOption() {
        if (mousedown) return
        yOffset -= optionToSelect.angle*friction/100;
        if ( Math.abs(optionToSelect.angle) > 0.5 && dragPosition && pickerDisplacement > maxScroll) {
            await asyncTimeout(8);
            centerSelectedOption();
        } else {
            resetDrag()
        }
    };

</script>

<svelte:window 
     on:mouseup|preventDefault|stopPropagation={releaseWheel}
     on:touchend|preventDefault|stopPropagation={releaseWheel}
     on:mousemove|preventDefault|stopPropagation={dragWheel}
/>

<div class="wheelWindow {classes}" role="button" tabindex="0"
     style="{fillParentStyle};
            padding: {marginY}% 0px;
            font-size: {fontSize}px; 
            line-height:{lineHeight}px; 
            overflow: {overflow};
            {style}; 
        "
     on:mousedown|preventDefault|stopPropagation={grabWheel}    
     on:touchstart|preventDefault|stopPropagation|nonpassive={grabWheel}
     on:touchmove|preventDefault|stopPropagation|nonpassive={dragWheel}
     on:mouseenter={() => {if (rollOnHover.enable) hovering = true}}
     on:mouseleave={() => {if (rollOnHover.enable && !mousedown && !touchstart) hovering = false}}
     bind:clientHeight={wheelWindowHeight} 
     bind:clientWidth={wheelWindowWidth}
>
    <div class="wheelWrapper" 
        style="
            {wheelWrapperScale};
            {hoverEnabled && stamp ? `transition: transform ${transformSpeed}s ease-in-out` : ""};
            transform-origin: {hoverEnabled && stamp ? "top" : "center"};
        "
        bind:clientHeight={wheelWrapperHeight} 
    >
        <div class="wheelContainer" 
            bind:clientHeight={wheelContainerHeight}
            style=" transform: {wheelContainerTransform};
                    height: {Math.max(wheelWrapperHeight, lineHeight*data.length)}px;
                    {!dragPosition ? `transition: transform ${transformSpeed}s ease-in-out` : ""};
                  "
        >
            {#each data as option, i (i)}
                <span class="wheelOption" 
                    style=" {wheelStyle[i]}; 
                            {i === optionToSelect.index  ? "opacity: 1" : ""};
                            {(hoverEnabled && !dragPosition) || !dragPosition ?`transition: transform ${transformSpeed}s ease-in-out, opacity ${transformSpeed}s ease-in-out` : ""}
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
	    font-family: Arial, -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu,
		Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
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