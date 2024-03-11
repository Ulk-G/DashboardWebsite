# Buoy Sensor Dashboard

This Svelte application provides a comprehensive dashboard for visualising sensor data from a buoy. It features live data updates for metrics such as roll, pitch, pressure, temperature, and humidity, alongside stability calculations such as draught and metacentric height.

## Getting Started

### Prerequisites

Ensure you have Node.js installed on your system to use npm for managing project dependencies.

### Installation

To set up this project locally, follow these steps:

1. Clone the repository to your local machine:

   ```bash
   git clone https://github.com/Ulk-G/DashboardWebsite
   cd DashboardWebsite
   ```

2. Install dependencies:

   ```bash
   npm i
   ```

   This will install all necessary dependencies as defined in `package.json`.

### Development

To run the application in development mode:

```bash
npm run dev -- --open
```

This command starts a local development server and opens the application in your default web browser. The server will automatically reload if you make edits to the source files.

### Production Build

To create an optimised production build:

```bash
npm run build
```

After building, you can preview the production version of your app by running:

```bash
npm run preview
```

### Connecting to Sensor Data

To receive live sensor data, ensure your buoy's serial device is connected to your computer. Use the "Connect to Serial Device" button within the dashboard to establish a connection.

## Project Structure

- `src/components`: Contains Svelte components, including `SensorGraph.svelte` for rendering sensor data charts.
- `src/routes`: SvelteKit's file-based routing. Your main dashboard page is defined here.
- `static`: Stores static assets like images.
- `package.json`: Defines project metadata and dependencies.

## Deploying

To deploy your app, consider using an adapter compatible with your target environment. Check the [SvelteKit documentation](https://kit.svelte.dev/docs/adapters) for more information on adapters.

## Contributing

Contributions are welcome! Please open an issue or submit a pull request with your proposed changes or additions.

## License

This project is licensed under the [Apache License](LICENSE).
