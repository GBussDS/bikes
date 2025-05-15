<script>
    import mapboxgl from "mapbox-gl";
    import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
    mapboxgl.accessToken = "pk.eyJ1IjoiZ2J1c3NkcyIsImEiOiJjbWFwaDMyMTgwaWs4MmxweWowdXc4a3NoIn0.Uw59GKHgA1j09CqvwCSAbg";

    import { onMount } from "svelte";

    let map;
    let stations = [];
    let mapViewChanged = 0;

    let departures = new Map();
    let arrivals;

    $: map?.on("move", evt => mapViewChanged++);

    async function initMap() {
        map = new mapboxgl.Map({
            container: 'map',
            center: [-71.09415, 42.36027],
            zoom: 12,
            style: "mapbox://styles/mapbox/streets-v12",
        });

        await new Promise(resolve => map.on("load", resolve));

        map.addSource("boston_route", {
            type: "geojson",
            data: "https://bostonopendata-boston.opendata.arcgis.com/datasets/boston::existing-bike-network-2022.geojson?outSR=%7B%22latestWkid%22%3A3857%2C%22wkid%22%3A102100%7D"
        });

        map.addLayer({
            id: "boston-bike-route",
            type: "line",
            source: "boston_route",
            paint: {
                "line-color": "#ff6600",
                "line-width": 4,
                "line-opacity": 0.8
            }
        });

        map.addSource("cambridge_route", {
            type: "geojson",
            data: "https://raw.githubusercontent.com/cambridgegis/cambridgegis_data/main/Recreation/Bike_Facilities/RECREATION_BikeFacilities.geojson"
        });

        map.addLayer({
            id: "cambridge-bike-route",
            type: "line",
            source: "cambridge_route",
            paint: {
                "line-color": "#00cc44",
                "line-width": 4,
                "line-opacity": 0.8
            }
        });
    }

    function getCoords (station) {
        let point = new mapboxgl.LngLat(+station.Long, +station.Lat);
        let {x, y} = map.project(point);
        return {cx: x, cy: y};
    }

    async function loadStationData() {
        try {
            const csvUrl = 'https://vis-society.github.io/labs/8/data/bluebikes-stations.csv';
            const data = await d3.csv(csvUrl);
            
            stations = data.map(station => ({
                id: station.Number,
                name: station.NAME,
                Lat: +station.Lat,
                Long: +station.Long,
            }));
            // console.log('Stations loaded:', stations.length);
        } catch (error) {
            console.error('Error loading station data:', error);
        }
    }

    async function loadStationDemand() {
        try {
            const csvUrl = 'https://vis-society.github.io/labs/8/data/bluebikes-traffic-2024-03.csv';
            const data = await d3.csv(csvUrl);
            
            trips = data.map(trip => {
                return {
                    id: trip.ride_id,
                    name: trip.NAME,
                    started_at: new Date(trip.started_at),
                    ended_at: new Date(trip.ended_at),
                    start_station_id: trip.start_station_id,
                    end_station_id: trip.end_station_id
                };
            });
            
            // console.log('Trips loaded:', trips.length);
        } catch (error) {
            console.error('Error loading station data:', error);
        }
    }

    
    function updateStations() {
        departures = d3.rollup(trips, v => v.length, d => d.start_station_id);
        arrivals = d3.rollup(trips, v => v.length, d => d.end_station_id);

        stations = stations.map(station => {
            let id = station.id;
            station.arrivals = arrivals.get(id) ?? 0;
            station.departures = departures.get(id) ?? 0;
            station.totalTraffic = station.arrivals + station.departures;
            return station;
        });
    }

    
    
    onMount(async () => {
        await initMap();
        await loadStationData();
        await loadStationDemand();

        updateStations();
    });

    
</script>
  
<main>
    <h1>Visualização Interativa do Tráfego de Bicicletas em Boston</h1>
    <p>
      Este projeto é uma visualização de mapa imersiva e interativa do tráfego de bicicletas
      na região de Boston em diferentes horários do dia. Usamos dados reais do sistema BlueBikes,
      com mais de 260 mil viagens registradas em março de 2024.
    </p>
    <p>
      O mapa mostra ruas e bairros da região, e permite navegar livremente com zoom e pan. 
      As estações são representadas por círculos, cujo tamanho indica o volume de tráfego, 
      e a cor mostra se há mais bicicletas chegando ou saindo.
    </p>
    <p>
      Você também poderá filtrar os dados por horário do dia com um controle deslizante, 
      e clicar em uma estação para visualizar isócronas de 5, 10, 15 e 20 minutos.
    </p>
</main>

<div id="map">
    {#key mapViewChanged}
        <svg>
            {#each stations as station}
                <circle { ...getCoords(station) } r="5" fill="steelblue" />
            {/each}
        </svg>
    {/key}
</div>

<style>
    @import url("$lib/global.css");
    #map {
        position: relative;
        width: 100%;
        height:600px;
    }
    #map svg {
        position: absolute;
        z-index: 1;
        width: 100%;
        height: 100%;
        pointer-events: none;
    }
    main {
      max-width: 800px;
      margin: 2rem auto;
      padding: 2rem;
      font-family: sans-serif;
      line-height: 1.6;
    }
  
    h1 {
      color: #1e88e5;
      font-size: 2.5rem;
      text-align: center;
    }
  
    p {
      font-size: 1.1rem;
      margin-top: 1rem;
    }
  </style>
  