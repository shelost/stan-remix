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
        }, 300) // Match the transition duration
      })
      
      card.on('dragstart', () => {
        // Disable transition during drag for immediate response
        el.style.transition = 'none'
      })
      
      card.on('dragmove', (e) => {
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
        
        // Reset flag for next drag
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

// Toggle fullscreen video expansion
const toggleExpand = () => {
  isExpanded.value = !isExpanded.value
}

const closeExpanded = () => {
  isExpanded.value = false
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
            :class="`stack-position-${card.stackIndex}`"
            :data-stack-index="card.stackIndex"
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
              @click="card.stackIndex === 0 && toggleExpand()"
            />
          </div>
        </TransitionGroup>
      </div>
    </main>

    <!-- Fullscreen Video Overlay -->
    <Transition name="expand">
      <div v-if="isExpanded" class="expanded-overlay" @click="closeExpanded">
        <video
          :src="currentVideo.src"
          class="expanded-video"
          autoplay
          loop
          muted
          playsinline
        />
        <button class="close-btn" @click.stop="closeExpanded">
          <svg width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5">
            <path d="M18 6L6 18M6 6l12 12"/>
          </svg>
        </button>
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

  // Fullscreen expanded video overlay
  .expanded-overlay {
    position: fixed;
    inset: 0;
    z-index: 1000;
    background: #000;
    display: flex;
    align-items: center;
    justify-content: center;
    cursor: pointer;

    .expanded-video {
      width: 100%;
      height: 100%;
      object-fit: contain;
    }

    .close-btn {
      position: absolute;
      top: 20px;
      right: 20px;
      width: 44px;
      height: 44px;
      border-radius: 50%;
      background: rgba(255, 255, 255, 0.2);
      backdrop-filter: blur(10px);
      border: none;
      color: white;
      cursor: pointer;
      display: flex;
      align-items: center;
      justify-content: center;
      transition: background 0.2s, transform 0.2s;

      &:hover {
        background: rgba(255, 255, 255, 0.3);
        transform: scale(1.1);
      }
    }
  }

  // Expand transition
  .expand-enter-active,
  .expand-leave-active {
    transition: opacity 0.3s ease, transform 0.3s cubic-bezier(0.34, 1.56, 0.64, 1);
  }

  .expand-enter-from,
  .expand-leave-to {
    opacity: 0;
    transform: scale(0.9);
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