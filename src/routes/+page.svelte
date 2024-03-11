<script>
    import SensorGraph from "../components/SensorGraph.svelte";

    // Website Variables
    let isConnected = false;
    let buoyFlippedTime = null;
    let buoyInDistress = false;
    let draftLineOffset = 10; // 10 (min) and -31 (max)
    let accumulatedData = [];
    let sensorData = {
      roll: 0,
      pitch: 0,
      pressure: 0,
      temperature: 0,
      humidity: 0
    };
    let expandedSensors = {
        roll: false,
        pitch: false,
        pressure: false,
        temperature: false,
        humidity: false
    };

    // Buoy Variables
    const WATER_DENSITY = 999.85       // kg/m^3 (@ 9°C)
    const GRAVITY = 9.81               // m/s^2
    const WHEEL_MASS = 1.515           // kg
    const WHEEL_VOLUME = 0.01052       // m^3
    const WHEEL_OUTER_DIAMETER = 0.3   // m
    const WHEEL_INNER_DIAMETER = 0.081 // m
    const WHEEL_THICKNESS = 0.19       // m
    const LB_TO_KG = 0.453592          // kg
    let plates = 0
    let draught = 0;
    let total_mass = 0;
    let submerged_volume = 0
    let kg = 0 // Keel to Centre of Gravity
    let kb = 0 // Keel to Centre of Buoyancy
    let momentOfInteria = 0.00039549476924410296   // Moment of Inertia (for group5 buoy specifically)
    let km = 0  // Keel to Metacentre
    let gm = 0  // Metacentric Height

    $: {
        total_mass = WHEEL_MASS + plates * LB_TO_KG // Total mass including plates
        submerged_volume = total_mass / WATER_DENSITY  // Volume needed to be submerged to float
        draught = WHEEL_THICKNESS * (((plates * LB_TO_KG) + WHEEL_MASS) / (WHEEL_VOLUME * WATER_DENSITY))
        
        // Baseline to Centre of Gravity  (0.04 is from baseline to the centre of mass of plates)
        kg = ((WHEEL_MASS * 0.345) + (plates * LB_TO_KG * 0.04)) / ((plates * LB_TO_KG) + WHEEL_MASS)

        // Baseline to Centre of Buoyancy
        kb = (draught / 2) + 0.26

        // Moment of Inertia
        //momentOfInteria = (Math.pi / 4) * (Math.pow(WHEEL_OUTER_DIAMETER / 2, 4) - Math.pow(WHEEL_INNER_DIAMETER / 2, 4));

        // Baseline to Metacentre
        km = momentOfInteria / submerged_volume

        // Metacentric Height
        gm = (kb + km) - kg
    }

    function toggleSensorExpansion(sensor) {
        expandedSensors[sensor] = !expandedSensors[sensor];
    }

    $: {
        const timestamp = new Date().toISOString();
        const date = new Date(timestamp);
        const options = {
            year: 'numeric',
            month: 'long',
            day: 'numeric',
            hour: '2-digit',
            minute: '2-digit',
            second: '2-digit',
            hour12: false
        };

        const formattedTime = date.toLocaleTimeString('en-UK', options);
        const milliseconds = date.getMilliseconds();
        const fullFormattedDate = `${formattedTime}.${milliseconds.toString().padStart(3, '0')}`;

        accumulatedData = [...accumulatedData, { fullFormattedDate, ...sensorData }];

        if (Math.abs(sensorData.roll) > 90 || Math.abs(sensorData.pitch) > 90) {
            if (!buoyFlippedTime) {
                buoyFlippedTime = new Date();
            }
        } else {
            buoyFlippedTime = null;
            buoyInDistress = false;
        }
        
        if (buoyFlippedTime && new Date() - buoyFlippedTime > 3000) {
            buoyInDistress = true;
        }
    }

    $: maxMagnitudeValue = Math.abs(sensorData.roll) > Math.abs(sensorData.pitch) ? sensorData.roll : sensorData.pitch;
    $: rotationStyle = `rotateZ(${-maxMagnitudeValue}deg)`;

    $: draftRotationStyle = `rotateZ(${-maxMagnitudeValue}deg)`;
    $: draftLineStyle = `transform: ${draftRotationStyle} translateY(${draftLineOffset - draught * (41 / WHEEL_THICKNESS)}px);`;

    async function connectToSerial() {
        if ("serial" in navigator) {
            try {
                const port = await navigator.serial.requestPort();
                await port.open({ baudRate: 9600 });
                const textDecoder = new TextDecoderStream();
                const readableStreamClosed = port.readable.pipeTo(textDecoder.writable);
                const reader = textDecoder.readable.getReader();

                isConnected = true;

                let buffer = '';

                while (port.readable) {
                    const { value, done } = await reader.read();

                    if (done) {
                        reader.releaseLock();
                        break;
                    }

                    buffer += value;

                    let newlineIndex = buffer.indexOf('\n');
                    while (newlineIndex !== -1) {
                        let jsonString = buffer.substring(0, newlineIndex).trim();
                        buffer = buffer.substring(newlineIndex + 1);

                        try {
                            const sensorReadings = JSON.parse(jsonString);
                            const timestamp = new Date().toISOString();
                            const date = new Date(timestamp);
                            const options = {
                                year: 'numeric',
                                month: 'long',
                                day: 'numeric',
                                hour: '2-digit',
                                minute: '2-digit',
                                second: '2-digit',
                                hour12: false
                            };

                            const formattedTime = date.toLocaleTimeString('en-UK', options);
                            const milliseconds = date.getMilliseconds();
                            const fullFormattedDate = `${formattedTime}.${milliseconds.toString().padStart(3, '0')}`;

                            sensorData = { ...sensorData, ...sensorReadings };
                            accumulatedData.push({ fullFormattedDate, ...sensorData });
                        } catch (e) {
                            console.error('Error parsing sensor data:', e);
                        }

                        // Look for the next newline character
                        newlineIndex = buffer.indexOf('\n');
                    }
                }
            } catch (err) {
                console.error('There was an error:', err);
                isConnected = false;
            }
        } else {
            console.error('Web Serial API not supported in this browser.');
        }
    }

    function saveAccumulatedDataToFile() {
        const jsonString = JSON.stringify(accumulatedData, null, 2);
        const blob = new Blob([jsonString], { type: 'application/json' });
        const url = URL.createObjectURL(blob);

        const link = document.createElement('a');
        link.href = url;
        link.download = 'accumulated_sensor_data.json';
        link.click();

        URL.revokeObjectURL(url);
    }
</script>
  
<main>
    <h1>Buoy Dashboard</h1>
    {#if buoyInDistress}
        <div class="warning-alert">
            <i class="fas fa-exclamation-triangle"></i>
            Warning: Buoy flipped and in distress!
        </div>
    {/if}
    <div class="content-container">
        <div class="dashboard">
        {#if isConnected}
            <div class="status connected">
                <i class="fas fa-circle"></i>
                <span>Connected to device</span>
            </div>
            <div class="sensor-data">
                <div class="data-item">
                    <i class="fas fa-weight-hanging"></i>
                    <label for="plates-input">Number of Plates:</label>
                    <input type="number" id="plates-input" min="0" max="10" bind:value={plates} />
                </div>
                <div class="data-item">
                    <i class="fas fa-compass"></i>
                    <span>Roll: {sensorData.roll} deg</span>
                    <button class="expand-btn" on:click={() => toggleSensorExpansion('roll')}>
                        {expandedSensors.roll ? 'Collapse' : 'Expand'}
                    </button>
                </div>
                <div class="data-item">
                    <i class="fas fa-compass"></i>
                    <span>Pitch: {sensorData.pitch} deg</span>
                    <button class="expand-btn" on:click={() => toggleSensorExpansion('pitch')}>
                        {expandedSensors.pitch ? 'Collapse' : 'Expand'}
                    </button>
                </div>
                <div class="data-item">
                    <i class="fas fa-tachometer-alt"></i>
                    <span>Pressure: {sensorData.pressure} kPa</span>
                    <button class="expand-btn" on:click={() => toggleSensorExpansion('pressure')}>
                        {expandedSensors.pressure ? 'Collapse' : 'Expand'}
                    </button>
                </div>
                <div class="data-item">
                    <i class="fas fa-thermometer-half"></i>
                    <span>Temp: {sensorData.temperature} °C</span>
                    <button class="expand-btn" on:click={() => toggleSensorExpansion('temperature')}>
                        {expandedSensors.temperature ? 'Collapse' : 'Expand'}
                    </button>
                </div>
                <div class="data-item">
                    <i class="fas fa-tint"></i>
                    <span>Humidity: {sensorData.humidity} %</span>
                    <button class="expand-btn" on:click={() => toggleSensorExpansion('humidity')}>
                        {expandedSensors.humidity ? 'Collapse' : 'Expand'}
                    </button>
                </div>
            </div>
            <div class="stability-data">
                <div class="data-item">
                    <span>Draught: {draught.toFixed(3)} m</span>
                </div>
                <div class="data-item">
                    <span>GM (Metacentric Height): {gm.toFixed(2)} m</span>
                </div>
                <div class="data-item">
                    <span>KB (Keel to Centre of Buoyancy): {kb.toFixed(2)} m</span>
                </div>
                <div class="data-item">
                    <span>KG (Keel to Centre of Gravity): {kg.toFixed(2)} m</span>
                </div>
                <div class="data-item">
                    <span>KM (Keel to Metacentre): {km.toFixed(2)} m</span>
                </div>
            </div>
        {:else}
            <button class="connect-btn" on:click={connectToSerial}>
                <i class="fas fa-plug"></i>
                Connect to Serial Device
            </button>
        {/if}
        </div>
        {#if isConnected}
            <div class="buoy-container">
                <div class="axis-container">
                    <div class="axis-line axis-x"></div>
                    <div class="axis-line axis-y"></div>
                    <div class="draft-line" style={draftLineStyle}></div>
                    <img src="buoy.png" alt="Buoy" style={`transform: ${rotationStyle};`} />
                </div>
            </div>
        {/if}
    </div>
    {#if expandedSensors.roll}
        <SensorGraph
            data={accumulatedData.map(item => ({ fullFormattedDate: item.fullFormattedDate, value: item.roll }))}
            label="Roll (deg)"
            color="rgba(75, 192, 192, 1)"
        />
    {/if}
    {#if expandedSensors.pitch}
        <SensorGraph
            data={accumulatedData.map(item => ({ fullFormattedDate: item.fullFormattedDate, value: item.pitch }))}
            label="Pitch (deg)"
            color="rgba(75, 192, 192, 1)"
        />
    {/if}
    {#if expandedSensors.pressure}
        <SensorGraph
            data={accumulatedData.map(item => ({ fullFormattedDate: item.fullFormattedDate, value: item.pressure }))}
            label="Pressure (kPA)"
            color="rgba(75, 192, 192, 1)"
        />
    {/if}
    {#if expandedSensors.temperature}
        <SensorGraph
            data={accumulatedData.map(item => ({ fullFormattedDate: item.fullFormattedDate, value: item.temperature }))}
            label="Temperature (°C)"
            color="rgba(75, 192, 192, 1)"
        />
    {/if}
    {#if expandedSensors.humidity}
        <SensorGraph
            data={accumulatedData.map(item => ({ fullFormattedDate: item.fullFormattedDate, value: item.humidity }))}
            label="Humidity (%)"
            color="rgba(75, 192, 192, 1)"
        />
    {/if}
    {#if isConnected}
        <button class="save-btn" on:click={saveAccumulatedDataToFile}>
            <i class="fas fa-save"></i>
            Save Accumulated Data
        </button>
    {/if}
</main>
  
<style>
    @import url('https://cdnjs.cloudflare.com/ajax/libs/font-awesome/5.15.3/css/all.min.css');
  
    main {
        max-width: 800px;
        margin: 0 auto;
        padding: 20px;
        font-family: Arial, sans-serif;
    }

    .content-container {
        display: flex;
        justify-content: space-between;
        align-items: center;
        margin-top: 40px;
    }
  
    h1 {
        text-align: center;
        color: #333;
    }
  
    .dashboard {
        background-color: #fff;
        border-radius: 10px;
        box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
        padding: 20px;
        flex: 1;
        margin-right: 40px;
        position: relative;
    }
  
    .buoy-container {
        width: 200px;
        height: 200px;
        display: flex;
        align-items: center;
        justify-content: center;
        background-color: #fff;
        border-radius: 10px;
        box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
    }

    .axis-container {
        position: relative;
        width: 80%;
        height: 80%;
        display: flex;
        align-items: center;
        justify-content: center;
    }

    .axis-line {
        position: absolute;
        background-color: #eee;
    }

    .axis-x {
        width: 100%;
        height: 1px;
    }

    .axis-y {
        width: 1px;
        height: 100%;
    }

    .draft-line {
        position: absolute;
        width: 100%;
        height: 2px;
        background-color: blue;
        transform-origin: center center;
        transition: transform 0.3s ease;
    }

    .buoy-container img {
        width: 60%;
        height: auto;
        transition: transform 0.3s ease;
    }

    .status {
        display: flex;
        align-items: center;
        margin-bottom: 20px;
        font-size: 18px;
    }
  
    .status i {
        margin-right: 10px;
    }
  
    .connected {
        color: #4caf50;
    }
  
    .sensor-data {
        display: grid;
        grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
        gap: 20px;
    }
  
    .data-item {
        display: flex;
        align-items: center;
        padding: 10px;
        background-color: #f5f5f5;
        border-radius: 5px;
    }
  
    .data-item i {
        margin-right: 10px;
        color: #333;
    }

    .data-item input[type="number"] {
        width: 60px;
        padding: 5px;
        border: 1px solid #ccc;
        border-radius: 5px;
        font-size: 14px;
        margin-left: 10px;
    }
  
    .connect-btn {
        display: block;
        width: 100%;
        padding: 10px;
        background-color: #2196f3;
        color: #fff;
        border: none;
        border-radius: 5px;
        font-size: 16px;
        cursor: pointer;
        transition: background-color 0.3s ease;
    }
  
    .connect-btn:hover {
        background-color: #1976d2;
    }
  
    .connect-btn i {
        margin-right: 10px;
    }

    .save-btn {
        display: block;
        width: 100%;
        padding: 10px;
        background-color: #4caf50;
        color: #fff;
        border: none;
        border-radius: 5px;
        font-size: 16px;
        cursor: pointer;
        transition: background-color 0.3s ease;
        margin-top: 20px;
    }

    .save-btn:hover {
        background-color: #45a049;
    }

    .save-btn i {
        margin-right: 10px;
    }

    .expand-btn {
        background-color: #2196f3;
        color: #fff;
        border: none;
        border-radius: 5px;
        padding: 5px 10px;
        font-size: 14px;
        cursor: pointer;
        margin-left: auto;
    }

    .expand-btn:hover {
        background-color: #1976d2;
    }

    .warning-alert {
        background-color: #ff4d4d;
        color: #fff;
        padding: 20px;
        border-radius: 5px;
        font-size: 18px;
        text-align: center;
        margin-top: 20px;
    }

    .warning-alert i {
        margin-right: 10px;
    }

    .stability-data {
        margin-top: 20px;
    }

    .stability-data .data-item {
        justify-content: space-between;
    }
  </style>
  