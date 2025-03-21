<script lang="ts">
    import { completedCourses, semesterClassRoutine, semesterName, User } from "$lib/store.svelte";
    import { fade, fly } from "svelte/transition";
    import { chooseColor, getDayNumber, resetColors, shorten, type ClassData } from "../classData.svelte";

    let mounted = $state(false);

    let classData: ClassData = $derived(semesterClassRoutine.value[semesterName.value]);

    //Day name today. Eg: Saturday
    let todaysDate = $state(new Date());
    let today = $derived(todaysDate.toLocaleString('en-us', {  weekday: 'long' }));
    
    let longestTimeEnd = $derived(getLongestTime(classData));
    let range: number[] = $derived(Array.from(Array(getLongestTime(classData)).keys()));

    let toRamadan = $state(false);
    const ramadanStartTime = new Date(2025, 1, 27).getTime();
    const ramadanEndTime = new Date(2025, 2, 30).getTime();
    let isRamadan = $derived(todaysDate.getTime() >= ramadanStartTime && todaysDate.getTime() <= ramadanEndTime);

    let completedSemesters = $derived.by(() => {
        let completed: Map<string, boolean> = new Map();
        for (const course in completedCourses.value) {
            const sem = completedCourses.value[course].semester;
            if (!completed.has(sem)){
                completed.set(sem, true);
            } 
        }

        return completed;
    });

    function convertClassTime(time: string): string {

        if (!isRamadan) return time;

        //semester must be current semester to show ramadan time
        const selectedSem = semesterName.value;
        const currentSem = Object.keys(semesterClassRoutine.value).at(completedSemesters.size);

        if (selectedSem != currentSem) return time;

        if (!toRamadan) return time;

        
        const timeMap: Record<string, string> = {
            //Lecture (1.5 hour) Reduced to 1 Hour
            "08:00 AM - 09:30 AM": "09:00 AM - 10:00 AM",
            "09:40 AM - 11:10 AM": "10:00 AM - 11:00 AM",
            "11:20 AM - 12:50 PM": "11:00 AM - 12:00 PM",
            "01:00 PM - 02:30 PM": "12:00 PM - 01:00 PM",
            "02:40 PM - 04:10 PM": "01:20 PM - 02:20 PM",
            "04:20 PM - 05:50 PM": "02:20 PM - 03:20 PM",
            //Lecture (2 hour) Reduced to 1:20Hrs
            "08:00 AM - 10:00 AM": "09:00 AM - 10:20 AM",
            "10:20 AM - 12:20 PM": "10:30 AM - 11:50 AM",
            "12:40 PM - 02:40 PM": "12:00 PM - 01:20 PM",
            "03:00 PM - 05:00 PM": "01:30 PM - 02:50 PM",
            //Lecture (1 hour) Reduced to 40 mins
            "08:00 AM - 09:00 AM": "09:00 AM - 09:40 AM",
            "09:10 AM - 10:10 AM": "09:40 AM - 10:20 AM",
            "10:20 AM - 11:20 AM": "10:20 AM - 11:00 AM",
            "11:30 AM - 12:30 PM": "11:10 AM - 11:50 AM",
            "12:40 PM - 01:40 PM": "12:00 PM - 12:40 PM",
            "01:50 PM - 02:50 PM": "12:40 PM - 01:20 PM",
            //Laboratory Class      Lab Class Reduced Timing 
            "08:00 AM - 10:20 AM": "09:00 AM - 10:30 AM",
            "10:20 AM - 12:40 PM": "10:30 AM - 12:00 PM",
            "12:40 PM - 03:00 PM": "12:00 PM - 01:30 PM",
            "03:00 PM - 05:20 PM": "01:30 PM - 03:00 PM",
            //Architecture Studio Class     Architecture Studio Reduced Timing
            "08:30 AM - 11:30 AM": "09:00 AM - 11:00 AM",
            "08:30 AM - 12:30 PM": "09:00 AM - 12:00 PM",
            "08:30 AM - 01:00 PM": "09:00 AM - 12:20 PM",
            "08:30 AM - 02:00 PM": "09:00 AM - 01:00 PM",
            "02:00 PM - 05:00 PM": "01:00 PM - 03:00 PM",
            //short classes 1.10 hours reduced to 1 hour
            "11:20 AM - 12:30 PM": "11:00 AM - 12:00 PM",
        }

        if (isRamadan) {
            return timeMap[time] || time;
        }

        return time;
    }


    function getLongestTime(classData: ClassData) {
        if (!classData) return 0;
        let longestTime = 0;
        for (const day in classData) {
            for (const time in classData[day]) {
                const timeEnd = timeParser(convertClassTime(time))[1];
                if (timeEnd > longestTime) longestTime = timeEnd;
            }
        }
        return Math.ceil((longestTime - 480) / 90) + 1;
    }

    function timeParser(timeRange: string): [number, number] {
        const times = timeRange.split("-").map((time) => time.trim());
        const startTime = parseTime(times[0]);
        const endTime = parseTime(times[1]);
        const startMinutes = startTime[0] * 60 + startTime[1];
        const endMinutes = endTime[0] * 60 + endTime[1];
        return [startMinutes, endMinutes];
    }

    function parseTime(timeStr: string): [number, number] {
        const match = RegExp(/(\d+):(\d+)\s+(AM|PM)/i).exec(timeStr);
        if (!match) throw new Error("Invalid time format");
        let hours = parseInt(match[1]);
        const minutes = parseInt(match[2]);
        const period = match[3].toUpperCase();
        if (hours === 12) {
            hours = period === "AM" ? 0 : 12;
        } else {
            hours += period === "PM" ? 12 : 0;
        }
        return [hours, minutes];
    }

    let debug = false;
    let svgParent = $state() as HTMLDivElement;

    $effect(() => {

        const rawCalMode = localStorage.getItem('calMode');
        if (!rawCalMode) {
            if (isRamadan) localStorage.setItem('calMode', "ramadan");
            else localStorage.setItem('calMode', "normal");
        }
        const calMode = rawCalMode ? rawCalMode === "ramadan" : false;

        toRamadan = calMode;

        mounted = true;

        const { svg } = createSVG();
        if (svgParent) {
            //remove all children
            while (svgParent.firstChild) {
                svgParent.removeChild(svgParent.firstChild);
            }
            svgParent.appendChild(svg);
        }
    });


    function createSVG() {
        const headerMargin = 50; // extra space for header
        const days = Object.entries(classData).sort((a, b) => getDayNumber(a[0]) - getDayNumber(b[0]));
        const numberOfDays = days.length;
        const topPadding = 60; // original top padding remains unchanged
        const dayGap = 5;
        const svgWidth = 60 + (numberOfDays * 120) + ((numberOfDays - 1) * dayGap);
        const svgHeight = topPadding + (longestTimeEnd * 90) + 20 + headerMargin; // increase overall height

        const imagePadding = 20;
        const scaleFactor = 4;

        const svgNS = "http://www.w3.org/2000/svg";
        const svg = document.createElementNS(svgNS, "svg");
        svg.setAttribute("viewBox", `0 0 ${svgWidth} ${svgHeight}`);
        svg.style.width = '100%';
        svg.style.height = `${svgHeight}px`;

        // Font settings
        const fontName = 'light';
        const fontUrl = 'fonts/now-web.woff2';
        const style = document.createElementNS(svgNS, 'style');
        style.textContent = `
            @font-face {
                font-family: '${fontName}';
                src: url('${fontUrl}');
            }
            text {
                font-family: '${fontName}', sans-serif;
            }
        `;
        svg.appendChild(style);

        // Draw background and header
        const background = document.createElementNS(svgNS, 'rect');
        background.setAttribute('x', '0');
        background.setAttribute('y', '0');
        background.setAttribute('width', `${svgWidth}`);
        background.setAttribute('height', `${svgHeight}`);
        background.setAttribute('fill', '#041e2f');
        svg.appendChild(background);


        // Header text for name and id of the user and the semester name
        //layout 
        // semester                 user id
        // semester                 user name

        const header = document.createElementNS(svgNS, 'g');
        header.setAttribute("transform", `translate(0, 0)`);
        const semesterText = document.createElementNS(svgNS, 'text');
        semesterText.setAttribute('x', '5');
        semesterText.setAttribute('y', '35');
        semesterText.setAttribute('fill', '#b8c4d0');
        semesterText.setAttribute('font-size', '15');
        semesterText.textContent = semesterName.value;
        header.appendChild(semesterText);

        const username = localStorage.getItem('UserName') || "xx-xxxxx-x";

        const userText = document.createElementNS(svgNS, 'text');
        userText.setAttribute('x', `${svgWidth - 5}`);
        userText.setAttribute('y', '20');
        userText.setAttribute('fill', '#708192');
        userText.setAttribute('font-size', '10');
        userText.setAttribute('text-anchor', 'end');
        userText.textContent = username; // user id
        header.appendChild(userText);

        const userName = User.value || "User Name";
        const userNameText = document.createElementNS(svgNS, 'text');
        userNameText.setAttribute('x', `${svgWidth - 5}`);
        userNameText.setAttribute('y', '35');
        userNameText.setAttribute('fill', '#b8c4d0');
        userNameText.setAttribute('font-size', '10');
        userNameText.setAttribute('text-anchor', 'end');
        userNameText.textContent = userName; // user name
        header.appendChild(userNameText);

        svg.appendChild(header);

        // Create a parent container for remaining elements
        const contentGroup = document.createElementNS(svgNS, 'g');
        contentGroup.setAttribute("transform", `translate(0, ${headerMargin})`);

        // Draw time markers and horizontal lines into contentGroup
        range.forEach((i) => {
            const y = topPadding + (i * 90);
            const line = document.createElementNS(svgNS, 'line');
            line.setAttribute('x1', '0');
            line.setAttribute('y1', `${y}`);
            line.setAttribute('x2', `${svgWidth}`);
            line.setAttribute('y2', `${y}`);
            line.setAttribute('stroke', '#5b72892e');
            line.setAttribute('stroke-width', '1.5');
            contentGroup.appendChild(line);

            const timeLabelBackground = document.createElementNS(svgNS, 'rect');
            timeLabelBackground.setAttribute('x', '0');
            timeLabelBackground.setAttribute('y', `${y-10}`);
            timeLabelBackground.setAttribute('width', '55');
            timeLabelBackground.setAttribute('height', '20');
            timeLabelBackground.setAttribute('fill', '#041e2f');
            contentGroup.appendChild(timeLabelBackground);

            const timeLabel = document.createElementNS(svgNS, 'text');
            timeLabel.setAttribute('x', '0');
            timeLabel.setAttribute('y', `${y+3}`);
            timeLabel.setAttribute('fill', '#708192');
            timeLabel.setAttribute('font-size', '10');
            timeLabel.textContent = i === 0 ? '8:00 am' : i === 1 ? '9:30 am' : i === 2 ? '11:00 am' : i === 3 ? '12:30 pm' : i === 4 ? '2:00 pm' : i === 5 ? '3:30 pm' : i === 6 ? '5:00 pm' : i === 7 ? '6:30 pm' : i === 8 ? '8:00 pm' : '9:30 pm';
            contentGroup.appendChild(timeLabel);
        });

        // Draw day columns and classes into contentGroup
        days.forEach(([day, classes], dayIndex) => {
            const dayX = 60 + (dayIndex * 120) + (dayIndex * dayGap);
            const dayName = document.createElementNS(svgNS, 'text');
            dayName.setAttribute('x', `${dayX + 60}`);
            dayName.setAttribute('y', `${topPadding - 28}`);
            dayName.setAttribute('text-anchor', 'middle');
            dayName.setAttribute('fill', '#708192');
            dayName.setAttribute('font-size', '14');
            dayName.setAttribute('font-weight', 'bold');
            dayName.textContent = day;
            contentGroup.appendChild(dayName);

            Object.entries(classes).forEach(([time, cls]) => {
                time = convertClassTime(time);
                const parsedTime = timeParser(time);
                const startY = topPadding + (parsedTime[0] - 479);
                const height = parsedTime[1] - parsedTime[0] - 1;
                const color = chooseColor(cls.class_id);

                const rect = document.createElementNS(svgNS, 'rect');
                rect.setAttribute('x', `${dayX}`);
                rect.setAttribute('y', `${startY}`);
                rect.setAttribute('width', '120');
                rect.setAttribute('height', `${height}`);
                rect.setAttribute('fill', color);
                rect.setAttribute('rx', '10');
                rect.setAttribute('ry', '10');
                contentGroup.appendChild(rect);

                const addText = (yOffset: number, content: string, size = '10') => {
                    const text = document.createElementNS(svgNS, 'text');
                    text.setAttribute('x', `${dayX + 60}`);
                    text.setAttribute('y', `${startY + yOffset}`);
                    text.setAttribute('fill', 'white');
                    text.setAttribute('font-size', size);
                    text.setAttribute('text-anchor', 'middle');
                    text.setAttribute('dominant-baseline', 'middle');
                    text.textContent = content;
                    contentGroup.appendChild(text);
                };
                addText(height / 2 - 15, shorten(cls.course_name) + ` [${cls.section}]`, "12");
                addText(height / 2, `Room: ${cls.room}`);
                addText(height / 2 + 15, time);
            });
        });

        // Append the group container to the SVG
        svg.appendChild(contentGroup);

        //write site name on the bottom right corner
        const siteURL = document.createElementNS(svgNS, 'text');
        siteURL.setAttribute('x', `${svgWidth - 5}`);
        siteURL.setAttribute('y', `${svgHeight - 5}`);
        siteURL.setAttribute('text-anchor', 'end');
        siteURL.setAttribute('fill', '#708192');
        siteURL.setAttribute('font-size', '10');
        siteURL.textContent = 'https://aiub.brainbird.org';
        svg.appendChild(siteURL);

        if (toRamadan) {
            //add ramadan png logo on the bottom left corner
            const ramadanLogo = document.createElementNS(svgNS, 'image');
            ramadanLogo.setAttribute('x', '5');
            ramadanLogo.setAttribute('y', `${svgHeight - 32}`);
            ramadanLogo.setAttribute('width', '32');
            ramadanLogo.setAttribute('height', '32');
            ramadanLogo.setAttributeNS('http://www.w3.org/1999/xlink', 'xlink:href', '/ramadan_logo.png')
            ramadanLogo.setAttribute('href', '/ramadan_logo.png');
            svg.appendChild(ramadanLogo);
            //add ramadan text on the bottom left corner
            const ramadanText = document.createElementNS(svgNS, 'text');
            ramadanText.setAttribute('x', '45');
            ramadanText.setAttribute('y', `${svgHeight - 15}`);
            ramadanText.setAttribute('fill', '#708192');
            ramadanText.setAttribute('font-size', '10');
            ramadanText.textContent = 'Ramadan';
            const ramadanText2 = document.createElementNS(svgNS, 'text');
            ramadanText2.setAttribute('x', '45');
            ramadanText2.setAttribute('y', `${svgHeight - 5}`);
            ramadanText2.setAttribute('fill', '#708192');
            ramadanText2.setAttribute('font-size', '10');
            ramadanText2.textContent = 'Kareem';
            svg.appendChild(ramadanText);
            svg.appendChild(ramadanText2);
        }

        return { svg, svgWidth, svgHeight, imagePadding, scaleFactor };
    }

    const convertImageToBase64 = async (url: string): Promise<string> => {
        const response = await fetch(url);
        const blob = await response.blob();
        return new Promise((resolve) => {
            const reader = new FileReader();
            reader.onloadend = () => resolve(reader.result as string);
            reader.readAsDataURL(blob);
        });
    };

    const save = async () => {
        if (!classData) return;

        // Create SVG
        const { svg, svgWidth, svgHeight, imagePadding, scaleFactor } = createSVG();

        // Convert the external image to Base64
        const imgElement = svg.querySelector('image'); // Find the <image> element inside SVG
        if (imgElement) {
            const imgUrl = imgElement.getAttribute('href') || imgElement.getAttribute('xlink:href');
            if (!imgUrl) return;
            const base64Image = await convertImageToBase64(imgUrl);
            imgElement.setAttribute('href', base64Image); // Replace with Base64
            imgElement.setAttribute('xlink:href', base64Image);
        }

        // Convert SVG to canvas
        const canvas = document.createElement('canvas');
        canvas.width = (svgWidth + 2 * imagePadding) * scaleFactor; // Scale canvas width
        canvas.height = (svgHeight + 2 * imagePadding) * scaleFactor; // Scale canvas height
        const ctx = canvas.getContext('2d') as CanvasRenderingContext2D;

        // Scale the drawing context
        ctx.scale(scaleFactor, scaleFactor);

        // Draw padding with the same background color as the SVG
        ctx.fillStyle = '#041e2f'; // Match this with the SVG background color
        ctx.fillRect(0, 0, canvas.width / scaleFactor, canvas.height / scaleFactor);

        // Create an image from the updated SVG
        const img = new Image();
        const svgData = new XMLSerializer().serializeToString(svg);
        img.src = 'data:image/svg+xml;base64,' + btoa(unescape(encodeURIComponent(svgData))); // Fix encoding issues

        // Wait for the image to load
        await new Promise((resolve) => {
            img.onload = resolve;
        });

        // Draw the SVG image onto the canvas with padding
        ctx.drawImage(img, imagePadding, imagePadding, svgWidth, svgHeight);

        // Save the canvas as an image
        const dataUrl = canvas.toDataURL('image/png');
        const a = document.createElement('a');
        a.href = dataUrl;
        a.download = `${User.value} ${semesterName.value}.png`;
        a.click();
    };


    function handleSelect(node: HTMLSelectElement){
        node.onchange = () => {
            localStorage.setItem('semester', node.value);
            resetColors();
            semesterName.value = node.value;
        }
    }

</script>

{#if debug}
    <div class="svgContainer" bind:this={svgParent} style="border: 2px solid"></div>
{/if}

{#if mounted}
<div class="wrapper" in:fade>
    {#if classData}
    <div class="dropdownlist" in:fly|global={{y: 10, duration: 200, delay: 100}}>
        <select use:handleSelect value={semesterName.value}>
            <option value="" selected disabled>Select Semester</option>
            {#if semesterClassRoutine.value}
                {#each Object.keys(semesterClassRoutine.value) as sem, i (sem)}
                    <option value={sem}>Semester: {i+1} - {sem}</option>
                {/each}
            {/if}
        </select>
        <button class="save" onclick={save} aria-label="Save Routine">
            <i class="fas fa-download"></i>
        </button>
        {#if isRamadan}
        <button class="toRamadan" aria-label="Ramadan Mode button" onclick={() => {
            toRamadan = !toRamadan;
            localStorage.setItem('calMode', toRamadan ? "ramadan" : "normal");
        }}
        style:color={toRamadan ? 'var(--accent)' : '#738789'}
        ><i class="fa-solid fa-mosque"></i></button>
        {/if}
    </div>
    <div class="classRoutine" out:fade={{duration: 50, delay: 0}}>
        {#key semesterName.value}
        <div class="timeline">
            {#each range as i (i)}
            <div class="time">
                <!-- Have am/pm -->
                    <div class="text" in:fly|global={{y: 10, delay: 50*(i+1)}}>
                        {i == 0 ? '8:00 am' : i == 1 ? '9:30 am' : i == 2 ? '11:00 am' : i == 3 ? '12:30 pm' : i == 4 ? '2:00 pm' : i == 5 ? '3:30 pm' : i == 6 ? '5:00 pm' : i == 7 ? '6:30 pm' : i == 8 ? '8:00 pm' : '9:30 pm'}
                    </div>
                </div>
                {/each}
            </div>
            <div class="routine">
            {#each Object.entries(classData).sort((a, b) => getDayNumber(a[0]) - getDayNumber(b[0])) as [day, classInfo], i (day)}
                {#if classInfo != null}
                <div class="day" style="height: {((longestTimeEnd * 90) - 90)}px;" class:focused={day == today} in:fly|global={{y: 10, delay: 50*(i+1)}}>
                    <div class="dayname">{day}</div>
                    {#each Object.entries(classInfo) as [time, Class], i}
                    {@const parsedTime = timeParser(convertClassTime(time))}
                    <div class="class" in:fly|global={{y: 10, delay: 10*i+1}} style="background: {chooseColor(Class.class_id)}; height: {(parsedTime[1] - parsedTime[0] - 1)}px; top: {(parsedTime[0] - 479)}px;">
                        <div class="toolTip">
                            {Class.course_name}
                        </div>
                        <div class="classContent">
                            <div class="coursename">{shorten(Class.course_name)} [{Class.section}]</div>
                            <div class="room">Room: {Class.room}</div>
                            <div class="time">{convertClassTime(time)}</div>
                        </div>
                    </div>
                    {/each}
                </div>
                {/if}
            {/each}
        </div>
        {/key}
    </div>
    {/if}
</div>
{/if}

<style lang="scss">

    .toRamadan {
        height: 100%;
        display: flex;
        color: white;
        padding: 5px;
        border-radius: 7px;
        margin-right: 3px;
        align-items: center;
        justify-content: center;
        gap: 5px;
    }

    .save {
        color: var(--accent);
        padding: 5px 16px;
        border-radius: 15px;
    }
    .wrapper{
        display: flex;
        gap: 10px;
        flex-direction: column;
        justify-content: center;
        align-items: center;
        padding: 15px;
        max-width: 100vw;
    }

    .coursename {
        font-size: 0.7rem;
    }
    
    .classRoutine{
        display: flex;
        gap: 5px;
        flex-direction: row;
        justify-content: flex-start;
        align-items: flex-start;
        padding: 60px 0px 20px 60px;
        height: 100%;
        position: relative;
        width: 100%;

        .routine{
            display: flex;
            gap: 5px;
            flex-direction: row;
            justify-content: flex-start;
            align-items: flex-start;
            padding: 0px;
            height: 100%;
            overflow-x: scroll;
            width: 100%;
            position: relative;
            scrollbar-width: none;
            padding-bottom: 20px;
            padding-top: 30px;

            .day{
                position: relative;
                overflow: visible;
                width: 120px;
                z-index: 10;

                &.focused{
                    background: #ffffff14;
                    border-radius: 5px;
                }
                .dayname{
                    text-align: center;
                    width: 120px;
                    font-weight: bold;
                    padding: 1px;
                    position: relative;
                    top: -28px;
                    color: var(--label-color);
                }
            }
        }
    }

    .timeline{
        padding-top: 30px;
        position: absolute;
        left: 0;
        width: 100%;
        display: flex;
        flex-direction: column;
        align-items: center;
        .time{
            display: flex;
            flex-direction: row;
            align-items: flex-start;
            justify-content: flex-start;
            position: relative;
            width: 100%;
            height: 100%;
            height: 90px;
            overflow: visible;
            font-size: 0.5rem;
            border-top: 2px solid #5b72892e;
            .text{
                position: relative;
                left: 0;
                margin-top: -15px;
                color: var(--label-color);
                background: var(--primary);
                padding: 5px;
                font-size: 0.7rem;
            }
        }
    }

    ::-webkit-scrollbar{
        display: none;
    }

    .class{
        display: flex;
        flex-direction: column;
        align-items: center;
        justify-content: center;
        gap: 5px;
        padding: 7px;
        border-radius: 10px;
        width: 100%;
        font-size: 0.7rem;
        position: absolute;
        cursor: pointer;

        .classContent{
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: flex-start;
            overflow: scroll;
            word-wrap: break-word;
        }

        .time, .room{
            font-size: 0.6rem;
        }

        &:hover{
            .toolTip{
                visibility: visible;
                opacity: 1;
                top: -20px;
            }
        }
    }

	.dropdownlist{
        background: var(--light-dark);
        border-radius: 10px;
        display: flex;
        flex-direction: row;
        align-items: center;
        justify-content: center;
        width: 100%;
	}

	
    select{
        width: 100%;
        height: 35px;
        background: var(--accent);
        outline: none;
        border: none;
        border-radius: 5px;
        color: aliceblue;
        text-align: center;
        cursor: pointer;
    }

</style>