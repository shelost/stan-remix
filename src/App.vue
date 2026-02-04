<script setup>
import { ref, computed, onMounted, onUnmounted, nextTick, watch } from 'vue'

const videos = [
  { 
    id: 1, 
    src: '/tikok_1.mp4', 
    title: 'My Stupid 2025 Goal', 
    pfp: '/pfp_abigail.jpeg',
    username: '@abigail',
    description: "My Stupid 2025 Goal",
    outperformance: 10.5
  },
  { 
    id: 2, 
    src: '/tiktok_2.mp4', 
    title: 'Watch this if youre lost...', 
    pfp: '/pfp_hothighpriestess.jpeg',
    username: '@hothighpriestess',
    description: "Right before you enter the greatest state of our life, watch for these signs...",
    outperformance: 8.2
  },
  { 
    id: 3, 
    src: '/tiktok_3.mp4', 
    title: 'You get used to how great your life is.', 
    pfp: '/pfp_steven.jpeg',
    username: '@steven',
    description: "You get used to how great your life is.",
    outperformance: 12.3
  },
  { 
    id: 4, 
    src: '/tiktok_4.mp4', 
    title: 'NBC Nightly News', 
    pfp: '/pfp_nbc.jpeg',
    username: '@nbcnews',
    description: "Federal Reserve Chair Jerome Powell says the Department of Justice has served subpoenas on the central bank...",
    outperformance: 9.7
  },
  { 
    id: 5, 
    src: '/tiktok_7.mp4', 
    title: 'This transition is 🔥 for any content creator #contentcreator #videotransition #instagramreels #reels #viral', 
    pfp: '/pfp_monty.jpeg',
    username: '@montylans',
    description: "This transition is 🔥 for any content creator #contentcreator #videotransition #instagramreels #reels #viral'",
    outperformance: 7.8
  },
  { 
    id: 6, 
    src: '/tiktok_5.mp4', 
    title: 'Over the past year...', 
    pfp: '/pfp_zohran.jpeg',
    username: '@zohran',
    description: "Over the past year, we have built something unprecedented in the history of our city. And we have done it together.",
    outperformance: 11.4
  }
]

// State - video queue for infinite loop
const videoQueue = ref([...videos])
const cards = ref([])
const stackEl = ref(null)
const videoRefs = ref({})
const isExpanded = ref(false)
const expandedVideoId = ref(null) // Track which video is expanded
const hasDragged = ref(false)
const isAnimating = ref(false)
const viewMode = ref('tinder') // 'tinder' or 'grid'
const isLayoutAnimating = ref(false)
let savedCardRect = null
let savedCardPositions = new Map() // For FLIP animation

// Swing references
let swingStack = null
let swingCards = []
let Direction = null
let Stack = null

// Current video computed (first in queue)
const currentVideo = computed(() => videoQueue.value[0] || videos[0])

// Expanded video computed (the video currently in fullscreen)
const expandedVideo = computed(() => {
  if (!expandedVideoId.value) return null
  return videos.find(v => v.id === expandedVideoId.value) || null
})

// Get current index for progress dots (find position in original videos array)
const currentIndex = computed(() => {
  const currentId = currentVideo.value?.id
  return videos.findIndex(v => v.id === currentId)
})

// Visible cards (show 3 at a time, stacked) - always from queue
const visibleCards = computed(() => {
  const result = []
  for (let i = 0; i < Math.min(3, videoQueue.value.length); i++) {
    result.push({
      ...videoQueue.value[i],
      stackIndex: i
    })
  }
  return result
})

// Initialize Swing on card elements
const initSwing = async () => {
  if (!stackEl.value || !Stack || !Direction) return
  
  // Destroy existing cards
  swingCards.forEach(card => card?.destroy?.())
  swingCards = []
  
  // Reset opacity and transform for all cards to ensure clean state
  const allCards = stackEl.value.querySelectorAll('.video-card')
  allCards?.forEach(el => {
    const videoId = parseInt(el.dataset.videoId)
    const stackPos = videoQueue.value.findIndex(v => v.id === videoId)
    if (stackPos >= 0 && stackPos < 3) {
      // Clear inline styles for visible cards
      el.style.opacity = ''
      el.style.transform = ''
      el.style.transition = ''
    }
  })
  
  // Create new stack
  swingStack = Stack({
    allowedDirections: [Direction.LEFT, Direction.RIGHT],
    throwOutConfidence: (xOffset, yOffset, element) => {
      const xConfidence = Math.min(Math.abs(xOffset) / (element.offsetWidth / 2), 1)
      return xConfidence
    },
    throwOutDistance: () => window.innerWidth * 1.2,
    rotation: (x) => x / 12,
    maxRotation: 25
  })
  
  // Wait for Vue to render
  await nextTick()
  
  // Get the top card element (only register the top card with Swing)
  const topCard = stackEl.value.querySelector('.video-card.stack-pos-0')
  if (!topCard) return
  
  const cardElements = [topCard]
  
  cardElements.forEach((el, idx) => {
    // Only make the top card draggable
    if (idx === 0) {
      const card = swingStack.createCard(el)
      swingCards.push(card)
      
      let isThrownOut = false
      
      card.on('throwout', (e) => {
        isThrownOut = true
        
        // Fade out the card smoothly
        el.style.transition = 'opacity 0.3s ease-out'
        el.style.opacity = '0'
        
        // Wait for fade animation to complete before updating queue
        setTimeout(() => {
          el.style.transition = 'none'
          el.style.transform = ''
          el.style.opacity = ''
          
          // Force reflow to apply the instant reset
          el.offsetHeight
          
          // Move swiped video to the back of the queue (infinite loop)
          const swipedVideo = videoQueue.value.shift()
          if (swipedVideo) {
            videoQueue.value.push(swipedVideo)
          }
          hasDragged.value = false
          
          // After Vue updates the DOM, ensure the new top card animates smoothly
          nextTick(() => {
            el.style.transition = ''
            
            const newTopCard = stackEl.value?.querySelector('.video-card.stack-pos-0')
            if (newTopCard && newTopCard !== el) {
              newTopCard.style.transition = 'transform 0.4s cubic-bezier(0.34, 1.56, 0.64, 1), opacity 0.3s ease'
            }
          })
        }, 300)
      })
      
      card.on('dragstart', () => {
        hasDragged.value = false
        el.style.transition = 'none'
      })
      
      card.on('dragmove', (e) => {
        hasDragged.value = true
        
        const likeEl = el.querySelector('.like-stamp')
        const nopeEl = el.querySelector('.nope-stamp')
        
        if (likeEl && nopeEl) {
          if (e.offset > 0) {
            likeEl.style.opacity = Math.min(e.offset / 100, 1)
            nopeEl.style.opacity = 0
          } else {
            nopeEl.style.opacity = Math.min(Math.abs(e.offset) / 100, 1)
            likeEl.style.opacity = 0
          }
        }
      })
      
      card.on('dragend', () => {
        // Only reset if card wasn't thrown out (user released without swiping far enough)
        if (!isThrownOut) {
          // Reset stamps
          const likeEl = el.querySelector('.like-stamp')
          const nopeEl = el.querySelector('.nope-stamp')
          if (likeEl) likeEl.style.opacity = 0
          if (nopeEl) nopeEl.style.opacity = 0
          
          // Reset card opacity if it was partially faded
          el.style.opacity = '1'
          
          // Re-enable transition for smooth snap-back
          el.style.transition = 'transform 0.4s cubic-bezier(0.34, 1.56, 0.64, 1), opacity 0.3s ease-out'
        }
        
        // Reset flag for next drag (will be checked by click handler if needed)
        // If no click happens, reset after a delay as fallback
        setTimeout(() => {
          hasDragged.value = false
        }, 200)
        
        isThrownOut = false
      })
    }
  })
}

// Play/pause videos based on current video
const updateVideoPlayback = async () => {
  await nextTick()
  
  // Find the top card's video (first in queue) and play it
  const topVideoId = currentVideo.value?.id
  if (topVideoId && videoRefs.value[topVideoId]) {
    videoRefs.value[topVideoId].play().catch(() => {})
  }
  
  // Pause all other videos
  Object.entries(videoRefs.value).forEach(([key, video]) => {
    if (!video) return
    const id = parseInt(key)
    if (id !== topVideoId) {
      video.pause()
      video.currentTime = 0
    }
  })
}

// Reinitialize when queue changes
watch(videoQueue, async () => {
  await nextTick()
  
  // Reset opacity for all visible cards (cards that were thrown out had opacity set to 0)
  const cardElements = stackEl.value?.querySelectorAll('.video-card')
  cardElements?.forEach(el => {
    const videoId = parseInt(el.dataset.videoId)
    const stackPos = videoQueue.value.findIndex(v => v.id === videoId)
    // Reset opacity for cards in visible positions (0, 1, 2)
    if (stackPos >= 0 && stackPos < 3) {
      el.style.opacity = ''
    }
  })
  
  // Short delay to let Vue update the DOM and CSS classes apply
  // Swing only applies transforms during drag, so it won't interfere with the CSS transition
  setTimeout(() => {
    if (viewMode.value === 'tinder') {
      initSwing()
    }
    updateVideoPlayback()
  }, 50)
}, { deep: true })

// Store video ref and autoplay if it's the top card
const setVideoRef = (el, id, stackIndex) => {
  if (el) {
    videoRefs.value[id] = el
    
    // Autoplay if this is the top card (stackIndex === 0)
    if (stackIndex === 0 && id === currentVideo.value?.id) {
      el.play().catch(() => {})
    } else {
      el.pause()
      el.currentTime = 0
    }
  }
}

// Lifecycle
onMounted(async () => {
  // Polyfill for Swing
  if (typeof window !== 'undefined' && !window.global) {
    window.global = window
  }
  
  // Import Swing
  const swing = await import('swing')
  Direction = swing.Direction
  Stack = swing.Stack
  
  await nextTick()
  
  // Delay to ensure DOM classes are fully applied
  setTimeout(() => {
    initSwing()
    updateVideoPlayback()
  }, 100)
})

onUnmounted(() => {
  swingCards.forEach(card => card?.destroy?.())
  swingStack = null
})

// Navigation - move last video to front (reverse swipe)
const goBack = () => {
  const lastVideo = videoQueue.value.pop()
  if (lastVideo) {
    videoQueue.value.unshift(lastVideo)
  }
}

// Handle video wrapper click - works for both layouts
const handleVideoWrapperClick = (videoId) => {
  // If already expanded, close it
  if (isExpanded.value) {
    closeExpanded()
    return
  }
  
  // If a drag occurred, don't expand (only relevant in tinder mode)
  if (hasDragged.value) {
    // Reset flag for next interaction
    hasDragged.value = false
    return
  }
  
  // In tinder mode, only allow top card to expand
  if (viewMode.value === 'tinder') {
    const topVideoId = videoQueue.value[0]?.id
    if (videoId !== topVideoId) return
  }
  
  toggleExpand(videoId)
}

// Toggle fullscreen video expansion with FLIP animation
const toggleExpand = async (videoId) => {
  if (isAnimating.value) return
  
  // Find the card element by video ID
  const card = stackEl.value?.querySelector(`.video-card[data-video-id="${videoId}"]`)
  if (!card) {
    console.warn('Card not found for video ID:', videoId)
    return
  }
  
  isAnimating.value = true
  expandedVideoId.value = videoId
  
  // Ensure the card is visible before capturing position
  card.style.opacity = '1'
  card.style.visibility = 'visible'
  
  // FLIP: First - save initial position
  savedCardRect = card.getBoundingClientRect()
  
  // Validate that we got a valid rect
  if (!savedCardRect || savedCardRect.width === 0 || savedCardRect.height === 0) {
    console.warn('Invalid card rect:', savedCardRect)
    isAnimating.value = false
    expandedVideoId.value = null
    return
  }
  
  // Apply expanded state immediately (no transition yet)
  card.style.transition = 'none'
  isExpanded.value = true
  
  await nextTick()
  
  // FLIP: Last - get final position
  const last = card.getBoundingClientRect()
  
  // Validate last rect
  if (!last || last.width === 0 || last.height === 0) {
    console.warn('Invalid last rect:', last)
    isExpanded.value = false
    isAnimating.value = false
    expandedVideoId.value = null
    return
  }
  
  // FLIP: Invert - calculate the transform to go from Last back to First
  const deltaX = savedCardRect.left - last.left + (savedCardRect.width - last.width) / 2
  const deltaY = savedCardRect.top - last.top + (savedCardRect.height - last.height) / 2
  const scaleX = savedCardRect.width / last.width
  const scaleY = savedCardRect.height / last.height
  
  // Apply inverse transform (element appears in original position)
  card.style.transform = `translate(${deltaX}px, ${deltaY}px) scale(${scaleX}, ${scaleY})`
  card.style.borderRadius = '24px'
  
  // Force reflow
  card.offsetHeight
  
  // FLIP: Play - animate to final position
  requestAnimationFrame(() => {
    card.style.transition = 'transform 0.5s cubic-bezier(0.32, 0.72, 0, 1), border-radius 0.5s cubic-bezier(0.32, 0.72, 0, 1)'
    card.style.transform = 'translate(0, 0) scale(1, 1)'
    card.style.borderRadius = '0'
    
    // Clean up after animation
    setTimeout(() => {
      card.style.transform = ''
      card.style.transition = ''
      card.style.borderRadius = ''
      isAnimating.value = false
    }, 500)
  })
}

const closeExpanded = async () => {
  if (isAnimating.value) return
  
  const videoId = expandedVideoId.value
  
  // If no expanded video or no saved rect, just reset state
  if (!videoId || !savedCardRect) {
    isExpanded.value = false
    expandedVideoId.value = null
    savedCardRect = null
    return
  }
  
  // Find the expanded card by video ID
  const card = stackEl.value?.querySelector(`.video-card[data-video-id="${videoId}"]`)
  if (!card) {
    isExpanded.value = false
    expandedVideoId.value = null
    savedCardRect = null
    return
  }
  
  isAnimating.value = true
  
  // FLIP: First - get current expanded position
  const first = card.getBoundingClientRect()
  
  // Remove expanded class (no transition yet)
  card.style.transition = 'none'
  isExpanded.value = false
  
  await nextTick()
  
  // Ensure card is visible after removing expanded class
  card.style.opacity = '1'
  card.style.visibility = 'visible'
  
  // FLIP: Last - the card is now in collapsed position
  // But we want to animate FROM expanded TO collapsed
  // So we apply a transform that makes it look like it's still expanded
  
  const deltaX = first.left - savedCardRect.left + (first.width - savedCardRect.width) / 2
  const deltaY = first.top - savedCardRect.top + (first.height - savedCardRect.height) / 2
  const scaleX = first.width / savedCardRect.width
  const scaleY = first.height / savedCardRect.height
  
  // Apply transform to make it appear at the expanded position
  card.style.transform = `translate(${deltaX}px, ${deltaY}px) scale(${scaleX}, ${scaleY})`
  card.style.borderRadius = '0'
  
  // Force reflow
  card.offsetHeight
  
  // FLIP: Play - animate back to original position
  requestAnimationFrame(() => {
    card.style.transition = 'transform 0.5s cubic-bezier(0.32, 0.72, 0, 1), border-radius 0.5s cubic-bezier(0.32, 0.72, 0, 1)'
    card.style.transform = ''
    card.style.borderRadius = ''
    
    // Clean up after animation
    setTimeout(() => {
      // Ensure card remains visible
      card.style.opacity = ''
      card.style.visibility = ''
      card.style.transition = ''
      savedCardRect = null
      expandedVideoId.value = null
      isAnimating.value = false
    }, 500)
  })
}

// Get stack position for a video
const getStackPosition = (videoId) => {
  const index = videoQueue.value.findIndex(v => v.id === videoId)
  return index >= 0 && index < 3 ? index : -1
}

// Card size constants (same for both layouts)
const CARD_WIDTH = 280
const CARD_HEIGHT = 440

// Get card style for tinder mode positioning
const getCardStyle = (videoId) => {
  const stackIndex = getStackPosition(videoId)
  if (stackIndex < 0 || stackIndex >= 3) {
    return { opacity: 0, pointerEvents: 'none' }
  }
  
  const scale = 1 - stackIndex * 0.04
  const translateX = stackIndex * 16
  const translateY = stackIndex * 16
  const opacity = 1 - stackIndex * 0.12
  
  return {
    transform: `scale(${scale}) translate(${translateX}px, ${translateY}px)`,
    zIndex: 10 - stackIndex,
    opacity: opacity.toString()
  }
}

// Calculate tinder stack positions (absolute positioning)
const getTinderPosition = (videoId) => {
  const stackIndex = getStackPosition(videoId)
  if (stackIndex < 0 || stackIndex >= 3) {
    return null
  }
  
  const containerRect = stackEl.value?.getBoundingClientRect()
  if (!containerRect) return null
  
  // Center the stack in the container
  const centerX = containerRect.width / 2 - CARD_WIDTH / 2
  const centerY = containerRect.height / 2 - CARD_HEIGHT / 2
  
  const scale = 1 - stackIndex * 0.04
  const offsetX = stackIndex * 16
  const offsetY = stackIndex * 16
  
  return {
    x: centerX + offsetX,
    y: centerY + offsetY,
    scale,
    opacity: 1 - stackIndex * 0.12,
    zIndex: 10 - stackIndex
  }
}

// Calculate grid positions
const getGridPosition = (videoIndex) => {
  const containerRect = stackEl.value?.getBoundingClientRect()
  if (!containerRect) return null
  
  const gap = 16
  const columns = 2
  const col = videoIndex % columns
  const row = Math.floor(videoIndex / columns)
  
  const scrollTop = stackEl.value?.parentElement?.scrollTop || 0
  
  // Account for search bar height (~60px)
  const searchBarHeight = viewMode.value === 'grid' ? 60 : 0
  
  return {
    x: col * (CARD_WIDTH + gap),
    y: searchBarHeight + row * (CARD_HEIGHT + 120 + gap) - scrollTop, // +120 for info section
    scale: 1,
    opacity: 1,
    zIndex: 1
  }
}

// FLIP animation for view mode switching
const switchViewMode = async (newMode) => {
  if (isLayoutAnimating.value || newMode === viewMode.value) return
  
  isLayoutAnimating.value = true
  savedCardPositions.clear()
  
  // Destroy Swing cards before transition
  if (swingStack) {
    swingCards.forEach(card => card?.destroy?.())
    swingCards = []
    swingStack = null
  }
  
  const cardElements = stackEl.value?.querySelectorAll('.video-card')
  
  // Reset opacity for all cards (important when switching to grid mode)
  // Cards that were thrown out in tinder mode had inline opacity:0
  if (newMode === 'grid') {
    cardElements?.forEach(el => {
      el.style.opacity = ''
    })
  }
  if (!cardElements || cardElements.length === 0) {
    viewMode.value = newMode
    isLayoutAnimating.value = false
    if (newMode === 'tinder') {
      await nextTick()
      initSwing()
    }
    return
  }
  
  // FIRST: Capture current positions using data-video-id
  cardElements.forEach((el) => {
    const rect = el.getBoundingClientRect()
    const videoId = parseInt(el.dataset.videoId)
    savedCardPositions.set(videoId, {
      x: rect.left,
      y: rect.top,
      width: rect.width,
      height: rect.height
    })
  })
  
  // Switch mode
  viewMode.value = newMode
  
  await nextTick()
  
  // LAST: Get new positions and INVERT
  cardElements.forEach((el) => {
    const videoId = parseInt(el.dataset.videoId)
    const first = savedCardPositions.get(videoId)
    if (!first) return
    
    const last = el.getBoundingClientRect()
    
    // Calculate deltas
    const deltaX = first.x - last.left
    const deltaY = first.y - last.top
    const deltaScaleX = first.width / last.width
    const deltaScaleY = first.height / last.height
    
    // Apply inverse transform (make it look like it's still at old position)
    el.style.transition = 'none'
    el.style.transform = `translate(${deltaX}px, ${deltaY}px) scale(${deltaScaleX}, ${deltaScaleY})`
    el.style.transformOrigin = 'top left'
  })
  
  // Force reflow
  stackEl.value?.offsetHeight
  
  // PLAY: Animate to final position
  requestAnimationFrame(() => {
    cardElements.forEach((el) => {
      el.style.transition = 'transform 0.5s cubic-bezier(0.32, 0.72, 0, 1)'
      el.style.transform = ''
    })
    
    // Clean up after animation
    setTimeout(async () => {
      cardElements.forEach((el) => {
        el.style.transition = ''
        el.style.transformOrigin = ''
      })
      savedCardPositions.clear()
      isLayoutAnimating.value = false
      
      // Reinitialize Swing if switching to tinder mode
      if (newMode === 'tinder') {
        await nextTick()
        initSwing()
      }
    }, 500)
  })
}
</script>

<template>
  <div class="app">
    <!-- Header -->
    <header class="header">
      <button class="back-btn" @click="goBack">
        <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round">
          <path d="M19 12H5M12 19l-7-7 7-7"/>
        </svg>
      </button>
    </header>

    <!-- View Mode Toggle -->
    <div class="view-mode-toggle" v-if="!isExpanded">
      <!-- Sliding pill indicator -->
      <div class="sliding-pill" :class="{ 'at-grid': viewMode === 'grid' }"></div>
      
      <button 
        class="mode-btn" 
        :class="{ active: viewMode === 'tinder' }"
        @click="switchViewMode('tinder')"
      >
        <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
          <rect x="3" y="3" width="18" height="18" rx="2"/>
        </svg>
      </button>
      <button 
        class="mode-btn" 
        :class="{ active: viewMode === 'grid' }"
        @click="switchViewMode('grid')"
      >
        <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
          <rect x="3" y="3" width="7" height="7"/>
          <rect x="14" y="3" width="7" height="7"/>
          <rect x="3" y="14" width="7" height="7"/>
          <rect x="14" y="14" width="7" height="7"/>
        </svg>
      </button>
    </div>

    <!-- Unified Video Container (morphs between layouts) -->
    <main class="video-container" :class="[`mode-${viewMode}`, { 'is-expanded': isExpanded }]">
      <!-- Search bar (grid mode only) -->

      <!-- Video cards container -->
      <div ref="stackEl" class="cards-wrapper">
        <div
          v-for="(video, index) in videos"
          :key="video.id"
          :data-video-id="video.id"
          class="video-card"
          :class="[
            { 'is-visible': viewMode === 'grid' || getStackPosition(video.id) >= 0 },
            { 'is-top': viewMode === 'tinder' && videoQueue[0]?.id === video.id },
            { 'expanded': isExpanded && expandedVideoId === video.id },
            `stack-pos-${getStackPosition(video.id)}`
          ]"
        >
          <!-- Video wrapper -->
          <div class="video-wrapper" @click.stop="handleVideoWrapperClick(video.id)">
            <video
              :ref="(el) => setVideoRef(el, video.id, getStackPosition(video.id))"
              :src="video.src"
              class="card-video"
              :autoplay="viewMode === 'tinder' && videoQueue[0]?.id === video.id"
              loop
              muted
              playsinline
            />
 
            
            <!-- Grid overlay -->
            <div class="grid-overlay" v-show="viewMode === 'grid' && !isExpanded">
              <span class="grid-title">{{ video.title }}</span>
            </div>
            
            <!-- Close button (expanded only) -->
            <button 
              v-if="expandedVideoId === video.id"
              class="expand-close-btn"
              :class="{ visible: isExpanded }"
              @click.stop="closeExpanded"
            >
              <svg width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5">
                <path d="M18 6L6 18M6 6l12 12"/>
              </svg>
            </button>
          </div>
          
          <!-- Grid info section -->
          <div class="grid-info" v-show="viewMode === 'grid'">
            <div class="grid-author-row">
              <div class="grid-author">
                <img :src="video.pfp" class="grid-avatar" />
                <span>{{ video.username.replace('@', '') }}</span>
              </div>
              <span class="grid-score">
                <svg width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5">
                  <path d="M7 17L17 7M17 7H7M17 7V17"/>
                </svg>
                {{ video.outperformance }}x
              </span>
            </div>
            <button class="grid-remix-btn">
              <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5">
                <path d="M3 12a9 9 0 1 0 9-9 9.75 9.75 0 0 0-6.74 2.74L3 8"/>
                <path d="M3 3v5h5"/>
              </svg>
              Remix
            </button>
          </div>
        </div>
      </div>
    </main>

    <!-- Backdrop overlay for closing expanded video -->
    <Transition name="fade">
      <div 
        v-if="isExpanded" 
        class="expanded-backdrop" 
        @click="closeExpanded"
      />
    </Transition>

    <!-- Fullscreen footer overlay -->
    <Transition name="slide-up">
      <div v-if="isExpanded && expandedVideo" class="fullscreen-footer" @click.stop>
        <div class="fullscreen-footer-content">
          <p class="fullscreen-description">{{ expandedVideo.description }}</p>
          
          <div class="fullscreen-meta">
            <span class="fullscreen-badge">
              <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5">
                <path d="M7 17L17 7M17 7H7M17 7V17"/>
              </svg>
              {{ expandedVideo.outperformance }}x Outperformer
            </span>
            <span class="fullscreen-author">
              <img :src="expandedVideo.pfp" :alt="expandedVideo.username" class="fullscreen-avatar" />
              {{ expandedVideo.username }}
            </span>
          </div>
          
          <button class="fullscreen-remix-btn">
            <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round">
              <path d="M3 12a9 9 0 1 0 9-9 9.75 9.75 0 0 0-6.74 2.74L3 8"/>
              <path d="M3 3v5h5"/>
            </svg>
            Remix
          </button>
        </div>
      </div>
    </Transition>

        
      <!-- Progress dots (Tinder mode only) -->
      <div v-if="viewMode === 'tinder'" class="progress">
        <span
          v-for="(v, i) in videos"
          :key="v.id"
          class="progress-dot"
          :class="{ active: i === currentIndex }"
        />
      </div>

    <!-- Info Panel (Tinder mode only) -->
    <footer v-if="viewMode === 'tinder'" class="footer">
      <Transition name="fade-slide" mode="out-in">
        <div :key="currentVideo.id" class="footer-content">
          <p class="description">{{ currentVideo.description }}</p>
          
          <div class="meta">
            <span class="badge">
              <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5">
                <path d="M7 17L17 7M17 7H7M17 7V17"/>
              </svg>
              {{ currentVideo.outperformance }}x Outperformer
            </span>
            <span class="author">
              <img :src="currentVideo.pfp" :alt="currentVideo.username" class="avatar" />
              {{ currentVideo.username }}
            </span>
          </div>
        </div>
      </Transition>
      
      <button class="remix-btn">
        <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round">
          <path d="M3 12a9 9 0 1 0 9-9 9.75 9.75 0 0 0-6.74 2.74L3 8"/>
          <path d="M3 3v5h5"/>
        </svg>
        Remix
      </button>
  
    </footer>

  </div>
</template>

<style lang="scss" scoped>
$font-family-inter: 'Inter', -apple-system, BlinkMacSystemFont, sans-serif;

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
  font-family: $font-family-inter;
}

.app {
  min-height: 100dvh;
  min-height: 100dvh;
  background: yellow;
  display: flex;
  flex-direction: column;
  font-family: $font-family-inter;
  color: #111;
  overflow: hidden;

  .header {
    padding: 16px 20px;
    flex-shrink: 0;
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    z-index: 50;

    .back-btn {
      width: 36px;
      height: 36px;
      border: none;
      background: none;
      cursor: pointer;
      color: #333;
      display: flex;
      align-items: center;
      justify-content: center;
      border-radius: 8px;
      transition: background 0.2s;

      &:hover {
        background: #f0f0f0;
      }
    }
  }

  // View mode toggle
  .view-mode-toggle {
    position: fixed;
    left: 20px;
    top: 50%;
    transform: translateY(-50%);
    display: flex;
    flex-direction: column;
    gap: 8px;
    z-index: 50;
    background: rgba(255, 255, 255, 0.9);
    backdrop-filter: blur(10px);
    padding: 6px;
    border-radius: 60px;
    box-shadow: 0 8px 32px rgba(0, 0, 0, 0.25);

    // Sliding pill behind active button
    .sliding-pill {
      position: absolute;
      width: 40px;
      height: 40px;
      background: #6355FF;
      border-radius: 40px;
      top: 6px; // Match padding
      left: 6px; // Match padding
      transition: transform 0.35s cubic-bezier(0.34, 1.56, 0.64, 1);
      z-index: 0;

      &.at-grid {
        transform: translateY(calc(100% + 8px)); // 40px button + 8px gap
      }
    }

    .mode-btn {
      position: relative;
      z-index: 1;
      width: 40px;
      height: 40px;
      border: none;
      background: transparent;
      border-radius: 12px;
      cursor: pointer;
      display: flex;
      align-items: center;
      justify-content: center;
      color: #999;
      transition: color 0.25s ease;

      &:hover:not(.active) {
        color: #666;
      }

      &.active {
        color: white;
      }
    }

    // Mobile: horizontal layout at top center
    @media (max-width: 768px) {
      left: 50%;
      top: 20px;
      transform: translateX(-50%);
      flex-direction: row;

      .sliding-pill {
        &.at-grid {
          transform: translateX(calc(100% + 8px)); // Move horizontally instead of vertically
        }
      }
    }
  }

  // Unified video container
  .video-container {
    flex: 1;
    display: flex;
    flex-direction: column;
    min-height: 0;
    overflow: visible;
    transition: all 0.5s cubic-bezier(0.32, 0.72, 0, 1);

    // Search bar for grid mode
    .grid-search {
      display: flex;
      align-items: center;
      gap: 12px;
      padding: 12px 16px;
      background: #f5f5f5;
      border-radius: 12px;
      margin: 0 16px 16px;
      opacity: 0;
      transform: translateY(-20px);
      transition: opacity 0.4s ease, transform 0.4s ease;
      
      svg {
        color: #999;
        flex-shrink: 0;
      }
      
      .search-input {
        flex: 1;
        border: none;
        background: transparent;
        font-size: 16px;
        color: #333;
        outline: none;
        
        &::placeholder {
          color: #999;
        }
      }
    }

    // Cards wrapper - switches between stack and grid
    .cards-wrapper {
      flex: 1;
      position: relative;
      transition: all 0.5s cubic-bezier(0.32, 0.72, 0, 1);
    }

    // Fixed card dimensions
    $card-width: 280px;
    $card-height: 440px;

    // Tinder mode styles
    &.mode-tinder {
      align-items: center;
      justify-content: center;
      padding: 0 20px;

      .cards-wrapper {
        width: $card-width + 40px; // Extra space for stack offset
        height: $card-height + 40px;
        position: relative;
        display: flex;
        align-items: center;
        justify-content: center;
      }

      .video-card {
        position: absolute;
        width: $card-width;
        height: $card-height;
        // Center the cards in the wrapper
        left: 50%;
        top: 50%;
        margin-left: -$card-width / 2;
        margin-top: -$card-height / 2;
        // Default transition for stack movement
        transition: transform 0.4s cubic-bezier(0.34, 1.56, 0.64, 1), opacity 0.3s ease;

        &:not(.is-visible) {
          opacity: 0;
          pointer-events: none;
        }

        // Stack position classes - these provide CSS-based positioning
        // that doesn't interfere with Swing's drag transforms
        &.stack-pos-0 {
          z-index: 10;
          opacity: 1;
          // No transform - let Swing control this card
        }

        &.stack-pos-1 {
          transform: scale(0.96) translate(16px, 16px);
          z-index: 9;
          opacity: 0.88;
        }

        &.stack-pos-2 {
          transform: scale(0.92) translate(32px, 32px);
          z-index: 8;
          opacity: 0.76;
        }

        .video-wrapper {
          width: 100%;
          height: 100%;
          border-radius: 24px;
          overflow: hidden;
          background: #fff;
          border: 2px solid white;
          box-shadow: 
            0 2px 8px rgba(0,0,0,0.08),
            0 8px 24px rgba(0,0,0,0.12);
          transition: border-radius 0.5s cubic-bezier(0.32, 0.72, 0, 1);
        }

        .grid-info {
          opacity: 0;
          height: 0;
          overflow: hidden;
          pointer-events: none;
        }
        
        .grid-overlay {
          opacity: 0;
          pointer-events: none;
        }
      }
    }

    // Grid mode styles
    &.mode-grid {
      overflow-y: auto;
      padding: 16px;
      padding-bottom: 24px;

      .grid-search {
        opacity: 1;
        transform: translateY(0);
        margin: 0 0 16px 0;
      }

      .cards-wrapper {
        display: grid;
        grid-template-columns: repeat(2, $card-width);
        gap: 16px;
        justify-items: center;
        width: fit-content;
        margin: auto;
      }

      .video-card {
        position: relative;
        display: flex;
        flex-direction: column;
        gap: 8px;
        opacity: 1 !important; // Ensure grid cards are always visible
        visibility: visible !important;
        pointer-events: auto;
        width: $card-width; 
        height: auto;

        .video-wrapper {
          width: $card-width;
          height: $card-height;
          border-radius: 16px;
          overflow: hidden;
          position: relative;
          background: #fff;
          box-shadow: 
            0 2px 8px rgba(0,0,0,0.08),
            0 4px 12px rgba(0,0,0,0.08);
          transition: border-radius 0.5s cubic-bezier(0.32, 0.72, 0, 1);
        }

        .grid-overlay {
          position: absolute;
          inset: 0;
          background: linear-gradient(to top, rgba(0,0,0,0.6) 0%, transparent 50%);
          display: flex;
          align-items: flex-end;
          padding: 12px;
          opacity: 1;
          transition: opacity 0.4s ease;

          .grid-title {
            color: white;
            font-size: 12px;
            font-weight: 600;
            line-height: 1.3;
            text-shadow: 0 1px 2px rgba(0,0,0,0.5);
          }
        }
        
        .grid-info {
          opacity: 1;
          height: auto;
          pointer-events: auto;
        }

        .like-stamp,
        .nope-stamp,
        .expand-close-btn {
          display: none;
        }
      }
    }

    // Video card base styles
    .video-card {
      cursor: grab;
      user-select: none;
      will-change: transform, opacity;

      &:active {
        cursor: grabbing;
      }

      &.expanded {
        position: fixed !important;
        top: 0 !important;
        left: 0 !important;
        right: 0 !important;
        bottom: 0 !important;
        width: 100vw !important;
        height: 100vh !important;
        margin: 0 !important; // Reset margins used for centering
        z-index: 1000 !important;
        cursor: default;
        // Note: Don't use transform: none !important here - it breaks FLIP animation

        .video-wrapper {
          width: 100%;
          height: 100%;
          border-radius: 0;
          border: none;
          background: #000;
          transition: border-radius 0.5s cubic-bezier(0.32, 0.72, 0, 1);
        }

        .card-video {
          object-fit: contain;
        }

        .like-stamp,
        .nope-stamp {
          display: none;
        }
        
        .grid-overlay,
        .grid-info {
          display: none;
        }
      }

      .card-video {
        width: 100%;
        height: 100%;
        object-fit: cover;
        display: block;
        background: #000;
      }

      // Close button
      .expand-close-btn {
        position: absolute;
        top: 20px;
        right: 20px;
        width: 44px;
        height: 44px;
        border-radius: 50%;
        background: rgba(255, 255, 255, 0.05);
        backdrop-filter: blur(10px);
        border: none;
        color: white;
        cursor: pointer;
        display: flex;
        align-items: center;
        justify-content: center;
        z-index: 1001;
        opacity: 0;
        pointer-events: none;
        transform: scale(0.8);
        transition: opacity 0.3s, transform 0.3s, background 0.2s;

        &.visible {
          opacity: 1;
          pointer-events: auto;
          transform: scale(1);
        }

        &:hover {
          background: rgba(255, 255, 255, 0.3);
          transform: scale(1.1);
        }
      }

      // Like/Nope stamps
      .like-stamp,
      .nope-stamp {
        position: absolute;
        top: 24px;
        font-size: 28px;
        font-weight: 800;
        padding: 8px 16px;
        border-radius: 8px;
        border: 4px solid;
        z-index: 10;
        opacity: 0;
        transform: rotate(-20deg);
        pointer-events: none;
        transition: opacity 0.1s;
      }

      .like-stamp {
        right: 24px;
        color: #22c55e;
        border-color: #22c55e;
        transform: rotate(20deg);
      }

      .nope-stamp {
        left: 24px;
        color: #ef4444;
        border-color: #ef4444;
      }
    }

    // Grid info styles
    .grid-info {
      padding: 0 4px;
      
      .grid-video-title {
        font-size: 14px;
        font-weight: 600;
        color: #111;
        margin-bottom: 2px;
        line-height: 1.3;
        display: -webkit-box;
        -webkit-line-clamp: 2;
        line-clamp: 2;
        -webkit-box-orient: vertical;
        overflow: hidden;
      }
      
      .grid-date {
        font-size: 12px;
        color: #999;
        margin-bottom: 8px;
      }
      
      .grid-author-row {
        display: flex;
        align-items: center;
        justify-content: space-between;
        margin: 8px 0 12px 0;
      }
      
      .grid-author {
        display: flex;
        align-items: center;
        gap: 6px;
        font-size: 14px;
        font-weight: 600;
        letter-spacing: -.25px;
        color: black;
        
        .grid-avatar {
          width: 20px;
          height: 20px;
          border-radius: 50%;
          object-fit: cover;
        }
      }
      
      .grid-score {
        display: inline-flex;
        align-items: center;
        gap: 4px;
        padding: 4px 8px;
        background: #6355FF;
        border-radius: 12px;
        font-size: 11px;
        font-weight: 600;
        color: white;
        
        svg {
          opacity: 0.9;
        }
      }
      
      .grid-remix-btn {
        width: 100%;
        display: flex;
        align-items: center;
        justify-content: center;
        gap: 6px;
        padding: 10px;
        background: transparent;
        border: 1.5px solid #6355FF;
        border-radius: 20px;
        color: #6355FF;
        font-size: 13px;
        font-weight: 600;
        cursor: pointer;
        transition: all 0.2s;
        
        &:hover {
          background: #6355FF;
          color: white;
        }
      }
    }
  }

  .footer {
    padding: 0px;
    padding-bottom: 40px;
    flex-shrink: 0;
    max-width: 380px;
    width: 100%;
    margin: 0 auto;
    position: relative;
    min-height: 120px; // Prevent layout shift during transitions

    // Footer content transition
    .fade-slide-enter-active {
      transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
    }

    .fade-slide-leave-active {
      transition: all 0.25s cubic-bezier(0.4, 0, 0.2, 1);
    }

    .fade-slide-enter-from {
      opacity: 0;
      transform: translateY(10px);
    }

    .fade-slide-leave-to {
      opacity: 0;
      transform: translateY(-10px);
    }

    .footer-content {
      width: 100%;
      height: 140px;
      margin: 12px 0;
    }

    .title {
      font-size: 18px;
      font-weight: 600;
      margin-bottom: 6px;
      letter-spacing: -0.5px;
    }

    .description {
      font-size: 16px;
      color: black;
      font-weight: 600;
      line-height: 1.2;
      white-space: pre-line;
      margin: 24px 0;
      letter-spacing: -0.4px;
    }

    .meta {
      display: flex;
      align-items: center;
      justify-content: space-between;
      margin-bottom: 0px;

      .badge {
        display: inline-flex;
        align-items: center;
        gap: 6px;
        padding: 8px 14px;
        background: #f3f4f6;
        border-radius: 24px;
        font-size: 13px;
        font-weight: 500;
        color: #333;
        letter-spacing: -0.32px;
      }

      .author {
        display: flex;
        align-items: center;
        gap: 8px;
        font-size: 14px;
        color: black;
        font-weight: 500;
        letter-spacing: -0.3px;


        .avatar {
          width: 28px;
          height: 28px;
          border-radius: 50%;
          object-fit: cover;
          display: block;
        }
      }
    }

    .remix-btn {
      width: 100%;
      display: flex;
      align-items: center;
      justify-content: center;
      gap: 10px;
      padding: 16px;
      background: #6355FF;
      border: none;
      border-radius: 40px;
      color: #fff;
      font-size: 16px;
      font-weight: 600;
      cursor: pointer;
      transition: transform 0.15s, box-shadow 0.15s;

      &:hover {
        transform: translateY(-2px);
        box-shadow: 0 8px 20px rgba(124, 58, 237, 0.35);
      }

      &:active {
        transform: translateY(0);
      }
    }

    .progress {
      display: flex;
      justify-content: center;
      gap: 6px;
      margin-top: 16px;

      .progress-dot {
        width: 8px;
        height: 8px;
        border-radius: 50%;
        background: #e5e5e5;
        transition: all 0.25s;

        &.active {
          background: #333;
          width: 24px;
          border-radius: 4px;
        }
      }
    }
  }

  // Backdrop overlay for expanded video
  .expanded-backdrop {
    position: fixed;
    inset: 0;
    z-index: 999;
    background: rgba(0, 0, 0, 0.9);
    cursor: pointer;
  }

  // Fullscreen footer overlay
  .fullscreen-footer {
    position: fixed;
    bottom: 0;
    left: 0;
    right: 0;
    z-index: 1001;
    padding: 180px 24px 32px 24px;
    background: linear-gradient(
      to top,
      rgba(0, 0, 0, 0.9) 0%,
      rgba(0, 0, 0, 0.7) 40%,
      rgba(0, 0, 0, 0) 100%
    );
    pointer-events: none;

    .fullscreen-footer-content {
      max-width: 420px;
      margin: 0 auto;
      pointer-events: auto;
    }

    .fullscreen-description {
      font-size: 16px;
      color: white;
      font-weight: 500;
      line-height: 1.4;
      margin-bottom: 16px;
      text-shadow: 0 1px 3px rgba(0, 0, 0, 0.5);
    }

    .fullscreen-meta {
      display: flex;
      align-items: center;
      justify-content: space-between;
      gap: 12px;
    }

    .fullscreen-badge {
      display: inline-flex;
      align-items: center;
      gap: 6px;
      padding: 8px 14px;
      background: rgba(255, 255, 255, 0.15);
      backdrop-filter: blur(10px);
      border-radius: 20px;
      font-size: 13px;
      font-weight: 600;
      color: white;

      svg {
        opacity: 0.9;
      }
    }

    .fullscreen-author {
      display: flex;
      align-items: center;
      gap: 8px;
      font-size: 14px;
      color: white;
      font-weight: 500;
    }

    .fullscreen-avatar {
      width: 32px;
      height: 32px;
      border-radius: 50%;
      object-fit: cover;
      border: 2px solid rgba(255, 255, 255, 0.3);
    }

    .fullscreen-remix-btn {
      width: 100%;
      display: flex;
      align-items: center;
      justify-content: center;
      gap: 10px;
      padding: 16px;
      margin-top: 16px;
      background: #6355FF;
      backdrop-filter: blur(10px);
      border: none;
      border-radius: 40px;
      color: #fff;
      font-size: 16px;
      font-weight: 600;
      cursor: pointer;
      transition: background 0.2s, transform 0.15s;

      &:hover {
        background: rgba(255, 255, 255, 0.3);
        transform: translateY(-2px);
      }

      &:active {
        transform: translateY(0);
      }
    }
  }

  // Slide up transition for fullscreen footer
  .slide-up-enter-active,
  .slide-up-leave-active {
    transition: transform 0.5s cubic-bezier(0.32, 0.72, 0, 1), opacity 0.4s ease;
  }

  .slide-up-enter-from,
  .slide-up-leave-to {
    transform: translateY(100%);
    opacity: 0;
  }

  // Fade transition for backdrop
  .fade-enter-active,
  .fade-leave-active {
    transition: opacity 0.4s ease;
  }

  .fade-enter-from,
  .fade-leave-to {
    opacity: 0;
  }

  @media (max-width: 400px) {
    .video-container.mode-tinder {
      .cards-wrapper {
        max-height: 380px;
      }
    }

    .footer {
      padding: 16px;
    }
  }
}
</style>