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
  let imageOverlay = $state(null);
  let showImageOverlay = $state(true);
  
  // Mapbox access token
  mapboxgl.accessToken = 'pk.eyJ1IjoiaW50ZWxsaXRlY2giLCJhIjoiY21jZTZzMm1xMHNmczJqcHMxOWtmaTd4aiJ9.rKhf7nuky9mqxxFAAIJlrQ';
  
  const mapStyles = [
    { value: 'mapbox://styles/mapbox/streets-v12', label: 'Streets' },
    { value: 'mapbox://styles/mapbox/satellite-v9', label: 'Satellite' },
    { value: 'mapbox://styles/mapbox/outdoors-v12', label: 'Outdoors' },
    { value: 'mapbox://styles/mapbox/light-v11', label: 'Light' },
    { value: 'mapbox://styles/mapbox/dark-v11', label: 'Dark' }
  ];
  
  const locations = [
    { name: 'St Joseph', lng: 120.9758, lat: 14.4716 },
  ];
  
  // Image overlay configuration
  const imageOverlayConfig = {
    imageUrl: 'https://docs.mapbox.com/mapbox-gl-js/assets/radar.gif',
    coordinates: [
      [120.9748, 14.4726], // top left
      [120.9768, 14.4726], // top right  
      [120.9768, 14.4706], // bottom right
      [120.9748, 14.4706]  // bottom left
    ]
  };
  
  const imageSizes = {
    'very-small': {
      coordinates: [
        [120.9753, 14.4721], // top left
        [120.9763, 14.4721], // top right  
        [120.9763, 14.4711], // bottom right
        [120.9753, 14.4711]  // bottom left
      ]
    },
    'small': {
      coordinates: [
        [120.9748, 14.4726], // top left
        [120.9768, 14.4726], // top right  
        [120.9768, 14.4706], // bottom right
        [120.9748, 14.4706]  // bottom left
      ]
    },
    'medium': {
      coordinates: [
        [120.9738, 14.4736], // top left
        [120.9778, 14.4736], // top right  
        [120.9778, 14.4696], // bottom right
        [120.9738, 14.4696]  // bottom left
      ]
    },
    'large': {
      coordinates: [
        [120.9708, 14.4766], // top left
        [120.9808, 14.4766], // top right  
        [120.9808, 14.4666], // bottom right
        [120.9708, 14.4666]  // bottom left
      ]
    }
  };
  
  let selectedSize = $state('small');
  
  onMount(() => {
    setTimeout(() => {
      initializeMap();
    }, 100);
    
    return () => {
      if (map) {
        map.remove();
        map = null;
      }
    };
  });
  
  function initializeMap() {
    if (!mapContainer) return;
    
    map = new mapboxgl.Map({
      container: mapContainer,
      style: mapStyle,
      center: [0, 20],
      zoom: 2,
      attributionControl: true,
      logoPosition: 'bottom-right'
    });
    
    map.on('load', () => {
      addLocationMarkers();
      addImageOverlay();
      map.resize();
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
      // Only add marker if not clicking on existing marker
      if (!e.originalEvent.target.closest('.mapboxgl-marker')) {
        addCustomMarker(e.lngLat.lng, e.lngLat.lat);
      }
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
  
  function addImageOverlay() {
    if (!map) return;
    
    const currentConfig = imageSizes[selectedSize];
    
    if (map.getSource('image-overlay')) {
      map.removeLayer('image-overlay-layer');
      map.removeSource('image-overlay');
    }
    
    map.addSource('image-overlay', {
      type: 'image',
      url: imageOverlayConfig.imageUrl,
      coordinates: currentConfig.coordinates
    });
    
    map.addLayer({
      id: 'image-overlay-layer',
      type: 'raster',
      source: 'image-overlay',
      paint: {
        'raster-opacity': 0.8,
        'raster-fade-duration': 300
      }
    });
    
    imageOverlay = true;
  }
  
  function updateImageSize(newSize) {
    selectedSize = newSize;
    if (map && map.getSource('image-overlay')) {
      addImageOverlay();
      if (!showImageOverlay) {
        map.setLayoutProperty('image-overlay-layer', 'visibility', 'none');
      }
    }
  }
  
  function toggleImageOverlay() {
    if (!map || !map.getSource('image-overlay')) return;
    
    showImageOverlay = !showImageOverlay;
    
    if (showImageOverlay) {
      map.setLayoutProperty('image-overlay-layer', 'visibility', 'visible');
    } else {
      map.setLayoutProperty('image-overlay-layer', 'visibility', 'none');
    }
  }
  
  function updateImageOpacity(opacity) {
    if (!map || !map.getSource('image-overlay')) return;
    
    map.setPaintProperty('image-overlay-layer', 'raster-opacity', opacity / 100);
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
    
    // Add event listener to the remove button after popup is opened
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
  
  function changeMapStyle(newStyle) {
    if (map && newStyle !== map.getStyle().name) {
      const currentMarkers = [...markers];
      const hadImageOverlay = map.getSource('image-overlay');
      
      map.setStyle(newStyle);
      
      map.once('styledata', () => {
        currentMarkers.forEach(marker => {
          if (marker.marker._map === null) {
            marker.marker.addTo(map);
          }
        });
        
        if (hadImageOverlay) {
          addImageOverlay();
          if (!showImageOverlay) {
            map.setLayoutProperty('image-overlay-layer', 'visibility', 'none');
          }
        }
        
        setTimeout(() => {
          map.resize();
        }, 100);
      });
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
  
  function flyToImageOverlay() {
    if (map) {
      map.flyTo({
        center: [120.9758, 14.4716],
        zoom: 16,
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
  }
  
  function resetView() {
    if (map) {
      map.flyTo({
        center: [0, 20],
        zoom: 2,
        speed: 1.5
      });
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
</svelte:head>

<main class="min-h-screen bg-gradient-to-br from-blue-50 to-indigo-100 p-4">
  <div class="max-w-7xl mx-auto">
    <!-- Header -->
    <div class="bg-gradient-to-r from-blue-600 to-purple-600 text-white rounded-t-xl p-6 shadow-lg">
      <h1 class="text-3xl font-bold text-center">Mapbox GL JS with Image Overlay</h1>
      <p class="text-center mt-2 opacity-90">Click on the map to add custom markers • Image overlay at St Joseph coordinates</p>
    </div>
    
    <!-- Controls -->
    <div class="bg-white p-6 border-x border-gray-200 shadow-sm">
      <div class="grid grid-cols-1 lg:grid-cols-5 gap-4">
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
        
        <!-- Image Overlay Controls -->
        <div>
          <label class="block text-sm font-semibold text-gray-700 mb-2">
            Image Overlay
          </label>
          <div class="flex flex-col gap-2">
            <button
              on:click={toggleImageOverlay}
              class="px-3 py-2 text-sm rounded-lg transition-colors duration-200 {showImageOverlay ? 'bg-green-500 hover:bg-green-600 text-white' : 'bg-gray-300 hover:bg-gray-400 text-gray-700'}"
            >
              {showImageOverlay ? 'Hide Overlay' : 'Show Overlay'}
            </button>
            <button
              on:click={flyToImageOverlay}
              class="px-3 py-2 bg-purple-500 text-white text-sm rounded-lg hover:bg-purple-600 transition-colors duration-200"
            >
              Go to Overlay
            </button>
          </div>
        </div>
        
        <!-- Opacity Control -->
        <div>
          <label class="block text-sm font-semibold text-gray-700 mb-2">
            Overlay Opacity
          </label>
          <input
            type="range"
            min="0"
            max="100"
            value="80"
            class="w-full h-2 bg-gray-200 rounded-lg appearance-none cursor-pointer"
            on:input={(e) => updateImageOpacity(e.target.value)}
          />
          <div class="text-xs text-gray-500 mt-1">0% - 100%</div>
        </div>
        
        <!-- Quick Locations -->
        <div>
          <label class="block text-sm font-semibold text-gray-700 mb-2">
            Quick Locations
          </label>
          <div class="flex flex-wrap gap-1">
            {#each locations.slice(0, 3) as location}
              <button
                on:click={() => flyToLocation(location)}
                class="px-2 py-1 bg-blue-500 text-white text-xs rounded-full hover:bg-blue-600 transition-colors duration-200 hover:scale-105"
              >
                {location.name}
              </button>
            {/each}
          </div>
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
              Clear Custom
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
          <h3 class="font-semibold text-gray-700 mb-1">Image Overlay</h3>
          <p class="text-xs {showImageOverlay ? 'text-green-600' : 'text-red-600'}">
            {showImageOverlay ? 'Visible' : 'Hidden'}
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
          <li>• <strong>Click</strong> anywhere on the map to add a red custom marker</li>
          <li>• <strong>Toggle image overlay</strong> to show/hide the raster image at St Joseph coordinates</li>
          <li>• <strong>Adjust opacity</strong> using the slider to make the overlay more or less transparent</li>
          <li>• <strong>Go to Overlay</strong> button will fly to the image overlay location</li>
          <li>• <strong>Use location buttons</strong> for quick navigation to major cities</li>
          <li>• <strong>Change map style</strong> - the overlay will persist across style changes</li>
        </ul>
      </div>
      
      <div class="mt-3 p-3 bg-yellow-50 rounded-lg">
        <h4 class="font-semibold text-yellow-800 mb-2">Image Overlay Info:</h4>
        <p class="text-xs text-yellow-700">
          <strong>Location:</strong> St Joseph (120.9758, 14.4716)<br>
          <strong>Current Image:</strong> Sample radar animation (replace with your subdivision map)<br>
          <strong>Coverage:</strong> Approximately 0.01° x 0.01° area - perfect for subdivision maps<br>
          <strong>Zoom Level:</strong> Optimized for detailed street-level view
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
  
  input[type="range"]::-webkit-slider-thumb {
    appearance: none;
    height: 20px;
    width: 20px;
    border-radius: 50%;
    background: #8b5cf6;
    cursor: pointer;
    box-shadow: 0 0 2px 0 #555;
    transition: background .15s ease-in-out;
  }
  
  input[type="range"]::-webkit-slider-thumb:hover {
    background: #7c3aed;
  }
  
  input[type="range"]::-moz-range-thumb {
    height: 20px;
    width: 20px;
    border-radius: 50%;
    background: #8b5cf6;
    cursor: pointer;
    border: none;
    box-shadow: 0 0 2px 0 #555;
    transition: background .15s ease-in-out;
  }
  
  input[type="range"]::-moz-range-thumb:hover {
    background: #7c3aed;
  }
</style>