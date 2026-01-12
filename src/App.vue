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
    src: '/tiktok_6.mp4', 
    title: 'I SAVED TIKTOK!', 
    pfp: '/pfp_trump.jpeg',
    username: '@realdonaldtrump',
    description: "I SAVED TIKTOK!",
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
const hasDragged = ref(false)
const isAnimating = ref(false)
let savedCardRect = null

// Swing references
let swingStack = null
let swingCards = []
let Direction = null
let Stack = null

// Current video computed (first in queue)
const currentVideo = computed(() => videoQueue.value[0] || videos[0])

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
  
  // Get all card elements and register with Swing
  const cardElements = stackEl.value.querySelectorAll('.swing-card')
  
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
          // Move swiped video to the back of the queue (infinite loop)
          const swipedVideo = videoQueue.value.shift()
          if (swipedVideo) {
            videoQueue.value.push(swipedVideo)
          }
          // Reset drag flag after queue update
          hasDragged.value = false
        }, 300) // Match the transition duration
      })
      
      card.on('dragstart', () => {
        // Reset drag flag at start of drag
        hasDragged.value = false
        // Disable transition during drag for immediate response
        el.style.transition = 'none'
      })
      
      card.on('dragmove', (e) => {
        // Mark that actual drag movement occurred
        hasDragged.value = true
        
        // Add visual feedback during drag
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
  
  // Short delay to let Vue update the DOM and CSS classes apply
  // Swing only applies transforms during drag, so it won't interfere with the CSS transition
  setTimeout(() => {
    initSwing()
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
  initSwing()
  
  // Auto-play top video after initialization
  setTimeout(() => {
    updateVideoPlayback()
  }, 200)
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

// Handle video click - only expand if it wasn't a drag
const handleVideoClick = () => {
  // If already expanded, close it
  if (isExpanded.value) {
    closeExpanded()
    return
  }
  
  // If a drag occurred, don't expand
  if (hasDragged.value) {
    // Reset flag for next interaction
    hasDragged.value = false
    return
  }
  
  toggleExpand()
}

// Toggle fullscreen video expansion with FLIP animation
const toggleExpand = async () => {
  if (isAnimating.value) return
  
  const topCard = stackEl.value?.querySelector('.swing-card.stack-position-0')
  if (!topCard) {
    isExpanded.value = true
    return
  }
  
  isAnimating.value = true
  
  // FLIP: First - save initial position
  savedCardRect = topCard.getBoundingClientRect()
  
  // Apply expanded state immediately (no transition yet)
  topCard.style.transition = 'none'
  isExpanded.value = true
  
  await nextTick()
  
  // FLIP: Last - get final position
  const last = topCard.getBoundingClientRect()
  
  // FLIP: Invert - calculate the transform to go from Last back to First
  const deltaX = savedCardRect.left - last.left + (savedCardRect.width - last.width) / 2
  const deltaY = savedCardRect.top - last.top + (savedCardRect.height - last.height) / 2
  const scaleX = savedCardRect.width / last.width
  const scaleY = savedCardRect.height / last.height
  
  // Apply inverse transform (element appears in original position)
  topCard.style.transform = `translate(${deltaX}px, ${deltaY}px) scale(${scaleX}, ${scaleY})`
  topCard.style.borderRadius = '24px'
  
  // Force reflow
  topCard.offsetHeight
  
  // FLIP: Play - animate to final position
  requestAnimationFrame(() => {
    topCard.style.transition = 'transform 0.5s cubic-bezier(0.32, 0.72, 0, 1), border-radius 0.5s cubic-bezier(0.32, 0.72, 0, 1)'
    topCard.style.transform = 'translate(0, 0) scale(1, 1)'
    topCard.style.borderRadius = '0'
    
    // Clean up after animation
    setTimeout(() => {
      topCard.style.transform = ''
      topCard.style.transition = ''
      topCard.style.borderRadius = ''
      isAnimating.value = false
    }, 500)
  })
}

const closeExpanded = async () => {
  if (isAnimating.value || !savedCardRect) return
  
  const topCard = stackEl.value?.querySelector('.swing-card.expanded')
  if (!topCard) {
    isExpanded.value = false
    return
  }
  
  isAnimating.value = true
  
  // FLIP: First - get current expanded position
  const first = topCard.getBoundingClientRect()
  
  // Remove expanded class (no transition yet)
  topCard.style.transition = 'none'
  isExpanded.value = false
  
  await nextTick()
  
  // FLIP: Last - the card is now in collapsed position
  // But we want to animate FROM expanded TO collapsed
  // So we apply a transform that makes it look like it's still expanded
  
  const deltaX = first.left - savedCardRect.left + (first.width - savedCardRect.width) / 2
  const deltaY = first.top - savedCardRect.top + (first.height - savedCardRect.height) / 2
  const scaleX = first.width / savedCardRect.width
  const scaleY = first.height / savedCardRect.height
  
  // Apply transform to make it appear at the expanded position
  topCard.style.transform = `translate(${deltaX}px, ${deltaY}px) scale(${scaleX}, ${scaleY})`
  topCard.style.borderRadius = '0'
  
  // Force reflow
  topCard.offsetHeight
  
  // FLIP: Play - animate back to original position
  requestAnimationFrame(() => {
    topCard.style.transition = 'transform 0.5s cubic-bezier(0.32, 0.72, 0, 1), border-radius 0.5s cubic-bezier(0.32, 0.72, 0, 1)'
    topCard.style.transform = ''
    topCard.style.borderRadius = ''
    
    // Clean up after animation
    setTimeout(() => {
      topCard.style.transition = ''
      savedCardRect = null
      isAnimating.value = false
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

    <!-- Card Stack -->
    <main class="main">
      <div ref="stackEl" class="stack">
        <TransitionGroup name="card-stack">
          <div
            v-for="card in visibleCards"
            :key="card.id"
            class="swing-card"
            :class="[
              `stack-position-${card.stackIndex}`,
              { expanded: isExpanded && card.stackIndex === 0 }
            ]"
            :data-stack-index="card.stackIndex"
            @click="isExpanded && card.stackIndex === 0 && closeExpanded()"
          >
            <!-- Like/Nope stamps -->
            <div class="like-stamp">LIKE</div>
            <div class="nope-stamp">NOPE</div>
            
            <!-- Video -->
            <video
              :ref="(el) => setVideoRef(el, card.id, card.stackIndex)"
              :src="card.src"
              class="card-video"
              :autoplay="card.stackIndex === 0"
              loop
              muted
              playsinline
              @click.stop="card.stackIndex === 0 && handleVideoClick()"
            />
            
            <!-- Close button (only visible when expanded) -->
            <button 
              v-if="card.stackIndex === 0"
              class="expand-close-btn"
              :class="{ visible: isExpanded }"
              @click.stop="closeExpanded"
            >
              <svg width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5">
                <path d="M18 6L6 18M6 6l12 12"/>
              </svg>
            </button>
          </div>
        </TransitionGroup>
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
      <div v-if="isExpanded" class="fullscreen-footer" @click.stop>
        <div class="fullscreen-footer-content">
          <p class="fullscreen-description">{{ currentVideo.description }}</p>
          
          <div class="fullscreen-meta">
            <span class="fullscreen-badge">
              <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5">
                <path d="M7 17L17 7M17 7H7M17 7V17"/>
              </svg>
              {{ currentVideo.outperformance }}x Outperformer
            </span>
            <span class="fullscreen-author">
              <img :src="currentVideo.pfp" :alt="currentVideo.username" class="fullscreen-avatar" />
              {{ currentVideo.username }}
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

        
      <!-- Progress dots -->
      <div class="progress">
        <span
          v-for="(v, i) in videos"
          :key="v.id"
          class="progress-dot"
          :class="{ active: i === currentIndex }"
        />
      </div>

    <!-- Info Panel -->
    <footer class="footer">
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
  min-height: 100vh;
  min-height: 100dvh;
  background: #fff;
  display: flex;
  flex-direction: column;
  font-family: $font-family-inter;
  color: #111;
  overflow: hidden;

  .header {
    padding: 16px 20px;
    flex-shrink: 0;

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

  .main {
    flex: 1;
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 0 20px;
    min-height: 0;

    .stack {
      position: relative;
      width: 100%;
      max-width: 340px;
      aspect-ratio: 9 / 14;
      max-height: 480px;

      // TransitionGroup move animation for cards shifting positions
      .card-stack-move {
        transition: transform 0.4s cubic-bezier(0.34, 1.56, 0.64, 1);
      }
      
      // Leave animation (fade out)
      .card-stack-leave-active {
        transition: opacity 0.3s ease-out;
        position: absolute;
      }
      
      .card-stack-leave-to {
        opacity: 0;
      }
      
      // Enter animation (for new cards coming in at the back)
      .card-stack-enter-active {
        transition: opacity 0.3s ease-out, transform 0.4s cubic-bezier(0.34, 1.56, 0.64, 1);
      }
      
      .card-stack-enter-from {
        opacity: 0;
      }

      .swing-card {
        position: absolute;
        inset: 0;
        background: #fff;
        border-radius: 24px;
        border: 2px solid white;
        overflow: hidden;
        box-shadow: 
          0 2px 8px rgba(0,0,0,0.08),
          0 8px 24px rgba(0,0,0,0.12);
        cursor: grab;
        user-select: none;
        will-change: transform, opacity;
        // Base transition (always present for all cards)
        transition: transform 0.4s cubic-bezier(0.34, 1.56, 0.64, 1), opacity 0.3s ease-out;

        &:active {
          cursor: grabbing;
        }
        
        // Stack position 0 (front card)
        &.stack-position-0 {
          transform: scale(1) translateX(0) translateY(0);
          z-index: 10;
          opacity: 1;
        }
        
        // Stack position 1
        &.stack-position-1 {
          transform: scale(0.96) translateX(20px) translateY(20px);
          z-index: 9;
          opacity: 0.88;
        }
        
        // Stack position 2
        &.stack-position-2 {
          transform: scale(0.92) translateX(40px) translateY(40px);
          z-index: 8;
          opacity: 0.76;
        }
        
        // Expanded fullscreen state
        &.expanded {
          position: fixed !important;
          top: 0 !important;
          left: 0 !important;
          right: 0 !important;
          bottom: 0 !important;
          width: 100vw !important;
          height: 100vh !important;
          max-width: none !important;
          max-height: none !important;
          z-index: 1000 !important;
          border: none !important;
          background: #000;
          cursor: default;
          // Don't set transform here - let FLIP animation handle it
          
          .card-video {
            object-fit: contain;
          }
          
          .like-stamp,
          .nope-stamp {
            display: none;
          }
        }
        
        // Close button inside card
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

        .card-video {
          width: 100%;
          height: 100%;
          object-fit: cover;
          display: block;
          background: #000;
          transition: .4 ease;
        }

        .card-dots {
          position: absolute;
          bottom: 12px;
          left: 50%;
          transform: translateX(-50%);
          display: flex;
          gap: 6px;
          z-index: 5;

          .dot {
            width: 6px;
            height: 6px;
            border-radius: 50%;
            border: none;
            background: rgba(255,255,255,0.5);
            cursor: pointer;
            padding: 0;

            &.active {
              background: #fff;
              width: 18px;
              border-radius: 3px;
            }
          }
        }

        .search-overlay {
          position: absolute;
          top: 12px;
          left: 12px;
          right: 12px;
          z-index: 5;

          .search-bar {
            display: flex;
            align-items: center;
            justify-content: space-between;
            background: rgba(255,255,255,0.9);
            backdrop-filter: blur(8px);
            padding: 10px 14px;
            border-radius: 24px;
            font-size: 13px;
            color: #666;
          }
        }
      }
    }
  }

  .footer {
    padding: 20px;
    padding-bottom: 60px;
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
      max-width: 500px;
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
    .main {
      .stack {
        max-height: 380px;
      }
    }

    .footer {
      padding: 16px;
    }
  }
}
</style>