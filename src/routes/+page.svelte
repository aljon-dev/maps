<!-- App.svelte -->
<script>
  import { onMount } from 'svelte';
  import mapboxgl from 'mapbox-gl';
  
  // Svelte 5 runes
  let mapContainer = $state();
  let map = $state();
  let mapStyle = $state('mapbox://styles/mapbox/streets-v12');
  let markers = $state([]);
  let coordinates = $state({ lng: 120.9758, lat: 14.4716 });
  let zoom = $state(2);
  let userLocation = $state(null);
  let userMarker = $state(null);
  let userLocationWatchId = $state(null);
  let routeLine = $state(null);
  let isTracking = $state(false);
  let isMapLoaded = $state(false);
  let isLoading = $state(false);
  let errorMessage = $state(null);
  let successMessage = $state(null);
  
  // Mapbox access token
  mapboxgl.accessToken = 'pk.eyJ1IjoiaW50ZWxsaXRlY2giLCJhIjoiY21jZTZzMm1xMHNmczJqcHMxOWtmaTd4aiJ9.rKhf7nuky9mqxxFAAIJlrQ';
  
  const mapStyles = [
    { value: 'mapbox://styles/mapbox/streets-v12', label: 'Mapbox Streets' },
    { value: 'mapbox://styles/mapbox/satellite-v9', label: 'Mapbox Satellite' },
    { value: 'mapbox://styles/mapbox/outdoors-v12', label: 'Mapbox Outdoors' },
    { value: 'mapbox://styles/mapbox/light-v11', label: 'Mapbox Light' },
    { value: 'mapbox://styles/mapbox/dark-v11', label: 'Mapbox Dark' },
    { value: 'osm', label: 'OpenStreetMap' }
  ];
  
  const locations = [
    { name: 'St Joseph', lng: 120.9758, lat: 14.4716 },
  ];

  onMount(() => {
    setTimeout(() => {
      initializeMap();
    }, 100);
    
    return () => {
      if (map) {
        map.remove();
        map = null;
      }
      stopTracking();
    };
  });
  
  function initializeMap() {
    if (!mapContainer) return;
    
    // Check if OSM style is selected
    const style = mapStyle === 'osm' ? {
      version: 8,
      sources: {
        'osm': {
          type: 'raster',
          tiles: ['https://a.tile.openstreetmap.org/{z}/{x}/{y}.png'],
          tileSize: 256,
          attribution: '© OpenStreetMap contributors',
          maxzoom: 19
        }
      },
      layers: [{
        id: 'osm-tiles',
        type: 'raster',
        source: 'osm',
        minzoom: 0,
        maxzoom: 22
      }]
    } : mapStyle;
    
    map = new mapboxgl.Map({
      container: mapContainer,
      style: style,
      center: [0, 20],
      zoom: 2,
      attributionControl: true,
      logoPosition: 'bottom-right'
    });

    map.on('load', () => {
      isMapLoaded = true;
      addLocationMarkers();
      map.resize();

      // Add the shared vector source
      map.addSource('custom-subdivision', {
        type: 'vector',
        url: 'mapbox://intellitech.cmcw5r4p80f841noymb6cadu3-3ftan'
      });

      // Polygon layer
      map.addLayer({
        id: 'custom-polygon-layer',
        type: 'fill',
        source: 'custom-subdivision',
        'source-layer': 'StartUp',
        paint: {
          'fill-color': '#3b82f6',
          'fill-opacity': 0.4
        },
        filter: ['==', '$type', 'Polygon']
      });

      // Line layer (on top of polygons)
      map.addLayer({
        id: 'custom-line-layer',
        type: 'line',
        source: 'custom-subdivision',
        'source-layer': 'StartUp',
        paint: {
          'line-color': '#ef4444',
          'line-width': 2
        },
        filter: ['==', '$type', 'LineString']
      });

      // Add navigation controls
      map.addControl(new mapboxgl.NavigationControl(), 'top-right');
    });
    
    map.on('mousemove', (e) => {
      coordinates = {
        lng: Number(e.lngLat.lng.toFixed(4)),
        lat: Number(e.lngLat.lat.toFixed(4))
      };
    });
    
    map.on('zoom', () => {
      zoom = Number(map.getZoom().toFixed(2));
    });
    
    map.on('click', (e) => {
      if (!e.originalEvent.target.closest('.mapboxgl-marker')) {
        addCustomMarker(e.lngLat.lng, e.lngLat.lat);
      }
    });
    
    map.on('error', (e) => {
      console.error('Map error:', e.error);
      showError('Map error: ' + e.error.message);
    });
    
    const resizeObserver = new ResizeObserver(() => {
      if (map) {
        map.resize();
      }
    });
    
    if (mapContainer) {
      resizeObserver.observe(mapContainer);
    }
  }

  function showError(message) {
    errorMessage = message;
    setTimeout(() => errorMessage = null, 5000);
  }

  function showSuccess(message) {
    successMessage = message;
    setTimeout(() => successMessage = null, 5000);
  }

  function changeMapStyle(newStyle) {
    if (!map || !isMapLoaded || newStyle === mapStyle) return;
    
    const currentMarkers = [...markers];
    const wasTracking = isTracking;
    
    if (wasTracking) stopTracking();
    
    // Handle OSM style differently
    const style = newStyle === 'osm' ? {
      version: 8,
      sources: {
        'osm': {
          type: 'raster',
          tiles: ['https://a.tile.openstreetmap.org/{z}/{x}/{y}.png'],
          tileSize: 256,
          attribution: '© OpenStreetMap contributors',
          maxzoom: 19
        }
      },
      layers: [{
        id: 'osm-tiles',
        type: 'raster',
        source: 'osm',
        minzoom: 0,
        maxzoom: 22
      }]
    } : newStyle;
    
    isMapLoaded = false;
    map.once('styledata', () => {
      isMapLoaded = true;
      
      // Re-add sources and layers
      if (!map.getSource('custom-subdivision')) {
        map.addSource('custom-subdivision', {
          type: 'vector',
          url: 'mapbox://intellitech.cmcvfdnh94n7d1opc2a1415bx-0v6mz'
        });
      }
      
      if (!map.getLayer('custom-polygon-layer')) {
        map.addLayer({
          id: 'custom-polygon-layer',
          type: 'fill',
          source: 'custom-subdivision',
          'source-layer': 'StartUp',
          paint: {
            'fill-color': '#3b82f6',
            'fill-opacity': 0.4
          },
          filter: ['==', '$type', 'Polygon']
        });
      }
      
      if (!map.getLayer('custom-line-layer')) {
        map.addLayer({
          id: 'custom-line-layer',
          type: 'line',
          source: 'custom-subdivision',
          'source-layer': 'StartUp',
          paint: {
            'line-color': '#ef4444',
            'line-width': 2
          },
          filter: ['==', '$type', 'LineString']
        });
      }
      
      // Re-add markers
      currentMarkers.forEach(marker => {
        if (marker.marker._map === null) {
          marker.marker.addTo(map);
        }
      });
      
      // Restart tracking if it was active
      if (wasTracking) startTracking();
    });
    
    map.setStyle(style);
    mapStyle = newStyle;
  }
  
  function navigateAlongLineString() {
    if (!map || !isMapLoaded) {
      showError('Map is not ready yet');
      return;
    }
    
    isLoading = true;
    
    try {
      // Get LineString features from the source
      const features = map.querySourceFeatures('custom-subdivision', {
        sourceLayer: 'StartUp',
        filter: ['==', '$type', 'LineString']
      });
      
      if (!features || features.length === 0) {
        showError('No LineString features found in the data source');
        isLoading = false;
        return;
      }
      
      // Use the first LineString feature
      const lineString = features[0];
      const coordinates = lineString.geometry.coordinates;
      
      // Remove existing route line if any
      if (routeLine) {
        if (map.getLayer('route-line')) map.removeLayer('route-line');
        if (map.getSource('route-line')) map.removeSource('route-line');
      }
      
      // Add the route line to the map
      map.addSource('route-line', {
        type: 'geojson',
        data: {
          type: 'Feature',
          properties: {},
          geometry: {
            type: 'LineString',
            coordinates: coordinates
          }
        }
      });
      
      map.addLayer({
        id: 'route-line',
        type: 'line',
        source: 'route-line',
        layout: {
          'line-join': 'round',
          'line-cap': 'round'
        },
        paint: {
          'line-color': '#3b82f6',
          'line-width': 4,
          'line-opacity': 0.7
        }
      });
      
      routeLine = true;
      
      // Calculate path distance
      const pathDistance = turf.length(turf.lineString(coordinates));
      const pathDuration = Math.min(Math.max(pathDistance * 50, 5000), 20000); // 5-20 seconds based on distance
      
      let startTime;
      const animateRoute = (currentTime) => {
        if (!startTime) startTime = currentTime;
        const elapsedTime = currentTime - startTime;
        const progress = Math.min(elapsedTime / pathDuration, 1);
        
        if (progress < 1) {
          const along = turf.along(
            turf.lineString(coordinates),
            pathDistance * progress,
            { units: 'meters' }
          );
          
          const point = along.geometry.coordinates;
          const bearing = turf.bearing(
            turf.point(coordinates[Math.max(0, Math.floor(progress * (coordinates.length - 1)))]),
            turf.point(coordinates[Math.min(coordinates.length - 1, Math.floor(progress * (coordinates.length - 1)) + 1)])
          );
          
          map.flyTo({
            center: point,
            bearing: bearing,
            zoom: 17,
            speed: 0.5,
            curve: 1,
            essential: true
          });
          
          requestAnimationFrame(animateRoute);
        } else {
          isLoading = false;
          showSuccess('Navigation completed successfully');
        }
      };
      
      requestAnimationFrame(animateRoute);
    } catch (error) {
      console.error('Navigation error:', error);
      showError('Error during navigation: ' + error.message);
      isLoading = false;
    }
  }
  
  function addLocationMarkers() {
    locations.forEach(location => {
      const marker = new mapboxgl.Marker({
        color: '#3b82f6',
        scale: 0.8
      })
        .setLngLat([location.lng, location.lat])
        .setPopup(
          new mapboxgl.Popup({ 
            offset: 25,
            closeButton: true,
            closeOnClick: false
          })
            .setHTML(`<div class="p-2">
                        <h3 class="font-bold text-gray-800">${location.name}</h3>
                        <p class="text-sm text-gray-600">Lng: ${location.lng}</p>
                        <p class="text-sm text-gray-600">Lat: ${location.lat}</p>
                      </div>`)
        )
        .addTo(map);
      
      markers = [...markers, {
        marker,
        lng: location.lng,
        lat: location.lat,
        id: `location-${location.name}`,
        isCustom: false
      }];
    });
  }
  
  function addCustomMarker(lng, lat) {
    const marker = new mapboxgl.Marker({
      color: '#ef4444',
      scale: 0.9,
      draggable: false
    })
      .setLngLat([lng, lat])
      .setPopup(
        new mapboxgl.Popup({ 
          offset: 25,
          closeButton: true,
          closeOnClick: false
        })
          .setHTML(`<div class="p-2">
                      <h3 class="font-bold text-gray-800">Custom Marker</h3>
                      <p class="text-sm text-gray-600">Lng: ${lng.toFixed(4)}</p>
                      <p class="text-sm text-gray-600">Lat: ${lat.toFixed(4)}</p>
                      <button id="remove-marker" 
                              class="mt-2 px-2 py-1 bg-red-500 text-white text-xs rounded hover:bg-red-600">
                        Remove
                      </button>
                    </div>`)
      )
      .addTo(map);
    
    const markerObj = {
      marker,
      lng,
      lat,
      id: `custom-${Date.now()}`,
      isCustom: true
    };
    
    markers = [...markers, markerObj];
    
    marker.getPopup().on('open', () => {
      const popupElement = marker.getPopup().getElement();
      const removeButton = popupElement?.querySelector('#remove-marker');
      
      if (removeButton) {
        removeButton.onclick = () => {
          removeMarker(markerObj.id);
          marker.getPopup().remove();
        };
      }
    });
  }
  
  function removeMarker(id) {
    const markerIndex = markers.findIndex(m => m.id === id);
    if (markerIndex !== -1) {
      markers[markerIndex].marker.remove();
      markers = markers.filter((_, index) => index !== markerIndex);
    }
  }
  
  function flyToLocation(location) {
    if (map) {
      map.flyTo({
        center: [location.lng, location.lat],
        zoom: 10,
        speed: 1.2,
        curve: 1.42,
        essential: true
      });
    }
  }
  
  function clearCustomMarkers() {
    const customMarkers = markers.filter(marker => marker.isCustom);
    customMarkers.forEach(marker => marker.marker.remove());
    markers = markers.filter(marker => !marker.isCustom);
    showSuccess('Custom markers cleared');
  }
  
  function resetView() {
    if (map) {
      map.flyTo({
        center: [0, 20],
        zoom: 2,
        speed: 1.5
      });
      showSuccess('View reset to default');
    }
  }
  
  // Live location functions
  function startTracking() {
    if (!navigator.geolocation) {
      showError('Geolocation is not supported by your browser');
      return;
    }
    
    if (userLocationWatchId) {
      stopTracking();
    }
    
    isTracking = true;
    
    // Create user marker if it doesn't exist
    if (!userMarker) {
      userMarker = new mapboxgl.Marker({
        color: '#10b981',
        scale: 1.2
      })
        .setLngLat([0, 0])
        .addTo(map);
    }
    
    userLocationWatchId = navigator.geolocation.watchPosition(
      (position) => {
        userLocation = {
          lng: position.coords.longitude,
          lat: position.coords.latitude,
          accuracy: position.coords.accuracy
        };
        
        if (userMarker) {
          userMarker.setLngLat([userLocation.lng, userLocation.lat]);
          
          // Center map on user location if zoomed out
          if (map.getZoom() < 10) {
            map.flyTo({
              center: [userLocation.lng, userLocation.lat],
              zoom: 14
            });
          }
        }
      },
      (error) => {
        console.error('Error getting location:', error);
        showError('Error getting location: ' + error.message);
        stopTracking();
      },
      {
        enableHighAccuracy: true,
        maximumAge: 10000,
        timeout: 5000
      }
    );
    
    showSuccess('Location tracking started');
  }
  
  function stopTracking() {
    if (userLocationWatchId) {
      navigator.geolocation.clearWatch(userLocationWatchId);
      userLocationWatchId = null;
    }
    isTracking = false;
    showSuccess('Location tracking stopped');
  }
  
  function toggleTracking() {
    if (isTracking) {
      stopTracking();
    } else {
      startTracking();
    }
  }
  
  $effect(() => {
    if (map && mapStyle) {
      changeMapStyle(mapStyle);
    }
  });
</script>

<svelte:head>
  <link href='https://api.mapbox.com/mapbox-gl-js/v3.0.1/mapbox-gl.css' rel='stylesheet' />
  <script src='https://unpkg.com/@turf/turf@6/turf.min.js'></script>
</svelte:head>

<main class="min-h-screen bg-gradient-to-br from-blue-50 to-indigo-100 p-4">
  <div class="max-w-7xl mx-auto">
    <!-- Header -->
    <div class="bg-gradient-to-r from-blue-600 to-purple-600 text-white rounded-t-xl p-6 shadow-lg">
      <h1 class="text-3xl font-bold text-center">LineString Navigation</h1>
      <p class="text-center mt-2 opacity-90">Live location tracking • Path navigation • GPS guidance</p>
    </div>
    
    <!-- Messages -->
    {#if errorMessage}
      <div class="bg-red-100 border-l-4 border-red-500 text-red-700 p-4 mb-4" role="alert">
        <p>{errorMessage}</p>
      </div>
    {/if}
    
    {#if successMessage}
      <div class="bg-green-100 border-l-4 border-green-500 text-green-700 p-4 mb-4" role="alert">
        <p>{successMessage}</p>
      </div>
    {/if}
    
    <!-- Controls -->
    <div class="bg-white p-6 border-x border-gray-200 shadow-sm">
      <div class="grid grid-cols-1 lg:grid-cols-4 gap-4">
        <!-- Map Style Selector -->
        <div>
          <label class="block text-sm font-semibold text-gray-700 mb-2">
            Map Style
          </label>
          <select 
            bind:value={mapStyle}
            class="w-full px-3 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent transition-all"
          >
            {#each mapStyles as style}
              <option value={style.value}>{style.label}</option>
            {/each}
          </select>
        </div>
        
        <!-- Live Location Tracking -->
        <div>
          <label class="block text-sm font-semibold text-gray-700 mb-2">
            Live Location
          </label>
          <button
            on:click={toggleTracking}
            class="w-full px-3 py-2 text-sm rounded-lg transition-colors duration-200 {isTracking ? 'bg-green-500 hover:bg-green-600 text-white' : 'bg-blue-500 hover:bg-blue-600 text-white'}"
          >
            {isTracking ? 'Stop Tracking' : 'Start Tracking'}
          </button>
        </div>
        
        <!-- Navigation Controls -->
        <div>
          <label class="block text-sm font-semibold text-gray-700 mb-2">
            Navigation
          </label>
          <button
            on:click={navigateAlongLineString}
            disabled={isLoading}
            class="w-full px-3 py-2 bg-purple-500 hover:bg-purple-600 text-white text-sm rounded-lg transition-colors duration-200 disabled:opacity-50 disabled:cursor-not-allowed"
          >
            {isLoading ? 'Navigating...' : 'Follow LineString'}
          </button>
        </div>
        
        <!-- Actions -->
        <div>
          <label class="block text-sm font-semibold text-gray-700 mb-2">
            Actions
          </label>
          <div class="flex flex-col gap-2">
            <button
              on:click={clearCustomMarkers}
              class="px-4 py-2 bg-red-500 text-white text-sm rounded-lg hover:bg-red-600 transition-colors duration-200"
            >
              Clear Custom Markers
            </button>
            <button
              on:click={resetView}
              class="px-4 py-2 bg-gray-500 text-white text-sm rounded-lg hover:bg-gray-600 transition-colors duration-200"
            >
              Reset View
            </button>
          </div>
        </div>
      </div>
    </div>
    
    <!-- Map Container -->
    <div class="relative bg-white border-x border-gray-200">
      <div 
        bind:this={mapContainer}
        class="w-full h-[500px] lg:h-[600px]"
        style="min-height: 400px;"
      ></div>
      
      <!-- Loading overlay -->
      {#if !map}
        <div class="absolute inset-0 bg-gray-100 flex items-center justify-center">
          <div class="text-center">
            <div class="animate-spin rounded-full h-12 w-12 border-b-2 border-blue-500 mx-auto mb-4"></div>
            <p class="text-gray-600">Loading map...</p>
          </div>
        </div>
      {/if}
    </div>
    
    <!-- Info Panel -->
    <div class="bg-white p-6 rounded-b-xl border-x border-b border-gray-200 shadow-lg">
      <div class="grid grid-cols-1 md:grid-cols-5 gap-4 text-sm">
        <div class="bg-gray-50 p-3 rounded-lg">
          <h3 class="font-semibold text-gray-700 mb-1">Mouse Position</h3>
          <p class="font-mono text-green-600">
            Lng: {coordinates.lng}
          </p>
          <p class="font-mono text-green-600">
            Lat: {coordinates.lat}
          </p>
        </div>
        
        <div class="bg-gray-50 p-3 rounded-lg">
          <h3 class="font-semibold text-gray-700 mb-1">Zoom Level</h3>
          <p class="font-mono text-blue-600 text-lg">{zoom}</p>
        </div>
        
        <div class="bg-gray-50 p-3 rounded-lg">
          <h3 class="font-semibold text-gray-700 mb-1">Total Markers</h3>
          <p class="font-mono text-purple-600 text-lg">{markers.length}</p>
        </div>
        
        <div class="bg-gray-50 p-3 rounded-lg">
          <h3 class="font-semibold text-gray-700 mb-1">Tracking</h3>
          <p class="text-xs {isTracking ? 'text-green-600' : 'text-red-600'}">
            {isTracking ? 'Active' : 'Inactive'}
          </p>
        </div>
        
        <div class="bg-gray-50 p-3 rounded-lg">
          <h3 class="font-semibold text-gray-700 mb-1">Map Style</h3>
          <p class="text-gray-600 text-xs">
            {mapStyles.find(s => s.value === mapStyle)?.label || 'Unknown'}
          </p>
        </div>
      </div>
      
      <div class="mt-4 p-3 bg-blue-50 rounded-lg">
        <h4 class="font-semibold text-blue-800 mb-2">Instructions:</h4>
        <ul class="text-xs text-blue-700 space-y-1">
          <li>• <strong>Click</strong> anywhere on the map to add a custom marker</li>
          <li>• <strong>Start Tracking</strong> to follow your live GPS location</li>
          <li>• <strong>Follow LineString</strong> to navigate along the path in your data</li>
          <li>• <strong>Change map style</strong> - includes OpenStreetMap option</li>
        </ul>
      </div>
      
      <div class="mt-3 p-3 bg-yellow-50 rounded-lg">
        <h4 class="font-semibold text-yellow-800 mb-2">Location Info:</h4>
        <p class="text-xs text-yellow-700">
          {#if userLocation}
            <strong>Your Location:</strong> {userLocation.lng.toFixed(6)}, {userLocation.lat.toFixed(6)}<br>
            <strong>Accuracy:</strong> {userLocation.accuracy?.toFixed(2) || 'Unknown'} meters<br>
          {:else}
            <strong>Your Location:</strong> Not tracking<br>
          {/if}
          <strong>Map Center:</strong> {coordinates.lng}, {coordinates.lat}<br>
          <strong>Zoom Level:</strong> {zoom}
        </p>
      </div>
    </div>
  </div>
</main>

<style>
  @import 'https://api.mapbox.com/mapbox-gl-js/v3.0.1/mapbox-gl.css';
  
  :global(.mapboxgl-popup-content) {
    border-radius: 0.75rem;
    box-shadow: 0 20px 25px -5px rgb(0 0 0 / 0.1), 0 10px 10px -5px rgb(0 0 0 / 0.04);
    padding: 0;
    max-width: 250px;
  }
  
  :global(.mapboxgl-popup-close-button) {
    color: rgb(107 114 128);
    font-size: 18px;
    padding: 8px;
  }
  
  :global(.mapboxgl-popup-close-button:hover) {
    color: rgb(55 65 81);
    background-color: rgb(243 244 246);
  }
  
  :global(.mapboxgl-popup-tip) {
    border-top-color: white;
  }
  
  :global(.mapboxgl-map) {
    border-radius: 0;
  }
  
  :global(.mapboxgl-ctrl-attrib) {
    background-color: rgba(255, 255, 255, 0.8);
    backdrop-filter: blur(4px);
  }
  
  :global(.mapboxgl-ctrl-zoom-in),
  :global(.mapboxgl-ctrl-zoom-out) {
    background-color: rgba(255, 255, 255, 0.9);
    backdrop-filter: blur(4px);
  }
</style>