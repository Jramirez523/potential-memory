# potential-memory
charity water

let highScore = 0
let level = 1
let nextLevelScore = 150
let timeLeft = 30
let gameRunning = false
let cleanCaught = 0

const gameArea = document.getElementById("gameArea")
const bucket = document.getElementById("bucket")
const scoreDisplay = document.getElementById("score")
const timerDisplay = document.getElementById("timer")
const levelDisplay = document.getElementById("level")
const highScoreDisplay = document.getElementById("highScore")

const overlay = document.getElementById("overlay")
const overlayTitle = document.getElementById("overlayTitle")
const overlayText = document.getElementById("overlayText")
const playAgainBtn = document.getElementById("playAgainBtn")
const soundToggleBtn = document.getElementById("soundToggleBtn")

let soundEnabled = true

highScore = parseInt(localStorage.getItem("rain-rescue-highscore") || "0", 10)
highScoreDisplay.textContent = highScore

const ACHIEVEMENTS_KEY = "rain-rescue-achievements"
const achievementsGrid = document.getElementById("achievementsGrid")

const ACHIEVEMENTS = [
  { id: "firstCatch", title: "First Catch", hint: "Catch your first clean drop", icon: "💧" },
  { id: "noMiss", title: "No Miss", hint: "Finish a round without missing a clean drop", icon: "🎯" },
  { id: "level3", title: "Level 3", hint: "Reach Level 3", icon: "🚀" },
  { id: "rainbowHunter", title: "Rainbow Hunter", hint: "Collect 5 rainbow drops", icon: "🌈" },
  { id: "highScore", title: "New High Score", hint: "Beat your previous high score", icon: "🏆" },
]

let unlockedAchievements = new Set(JSON.parse(localStorage.getItem(ACHIEVEMENTS_KEY) || "[]"))
let cleanMissed = false
let rainbowCount = 0

function initAchievements() {
  achievementsGrid.innerHTML = ""
  for (const achievement of ACHIEVEMENTS) {
    const badge = document.createElement("div")
    badge.className = `badge ${unlockedAchievements.has(achievement.id) ? "unlocked" : "locked"}`
    badge.id = `ach-${achievement.id}`
    badge.innerHTML = `
      <div class="icon">${achievement.icon}</div>
      <div class="title">${achievement.title}</div>
      <div class="hint">${achievement.hint}</div>
    `
    achievementsGrid.appendChild(badge)
  }
}

function unlockAchievement(id) {
  if (unlockedAchievements.has(id)) return
  unlockedAchievements.add(id)
  localStorage.setItem(ACHIEVEMENTS_KEY, JSON.stringify(Array.from(unlockedAchievements)))
  const badge = document.getElementById(`ach-${id}`)
  if (badge) {
    badge.classList.remove("locked")
    badge.classList.add("unlocked")
    badge.classList.add("pulse")
    setTimeout(() => badge.classList.remove("pulse"), 950)
  }
}

initAchievements()

soundToggleBtn.addEventListener("click", () => {
  soundEnabled = !soundEnabled
  soundToggleBtn.textContent = `Sound: ${soundEnabled ? "On" : "Off"}`
})

let drops = []
let lastSpawnTime = 0
let spawnInterval = 900
let animationId = null
let lastUpdateTime = 0

const GAME_DURATION = 30
const DROP_SIZE = 40

// UI bindings

document.getElementById("startBtn").addEventListener("click", () => startGame())
document.getElementById("resetBtn").addEventListener("click", resetGame)
playAgainBtn.addEventListener("click", () => {
  hideOverlay()
  startGame()
})

document.addEventListener("mousemove", (event) => {
  if (!gameRunning) return
  const rect = gameArea.getBoundingClientRect()
  const x = event.clientX - rect.left
  moveBucketTo(x)
})

document.addEventListener("keydown", (event) => {
  if (!gameRunning) return
  const step = 18
  const bucketRect = bucket.getBoundingClientRect()
  const parentRect = gameArea.getBoundingClientRect()

  if (event.key === "ArrowLeft" || event.key === "a") {
    moveBucketTo(bucketRect.left - parentRect.left - step)
  }
  if (event.key === "ArrowRight" || event.key === "d") {
    moveBucketTo(bucketRect.left - parentRect.left + step)
  }
})

function startGame() {
  if (gameRunning) return

  gameRunning = true
  score = 0
  level = 1
  nextLevelScore = 150
  timeLeft = GAME_DURATION
  spawnInterval = 900
  drops = []
  lastSpawnTime = 0

  scoreDisplay.textContent = score
  levelDisplay.textContent = level
  timerDisplay.textContent = timeLeft
  cleanCaught = 0
  rainbowCount = 0
  cleanMissed = false
  updateMilestoneDisplay()

  // Reset game area and prepare bucket
  gameArea.querySelectorAll(".drop").forEach((d) => d.remove())
  bucket.style.left = "50%"
  bucket.style.transform = "translateX(-50%)"

  playSound("start")
  hideOverlay()

  lastUpdateTime = performance.now()
  animationId = requestAnimationFrame(gameLoop)
  startTimer()
}

function gameLoop(timestamp) {
  if (!gameRunning) return

  const delta = timestamp - lastUpdateTime
  lastUpdateTime = timestamp

  if (timestamp - lastSpawnTime > spawnInterval) {
    spawnDrop()
    lastSpawnTime = timestamp

    // Gradually speed up as the level grows
    spawnInterval = Math.max(250, spawnInterval - 5 - level)
  }

  updateDrops(delta)
  animationId = requestAnimationFrame(gameLoop)
}

function startTimer() {
  const timer = setInterval(() => {
    if (!gameRunning) {
      clearInterval(timer)
      return
    }

    timeLeft -= 1
    timerDisplay.textContent = timeLeft

    if (timeLeft <= 0) {
      clearInterval(timer)
      endGame()
    }
  }, 1000)
}

function spawnDrop() {
  const drop = document.createElement("div")
  drop.classList.add("drop")

  const typeRoll = Math.random()
  let type = "clean"

  // Increase difficulty by shifting probabilities each level
  const badBase = 0.26 + (level - 1) * 0.03
  const bonusBase = 0.18
  const rainbowBase = 0.07
  const cleanMax = 0.62 - (level - 1) * 0.03

  if (typeRoll < cleanMax) {
    type = "clean"
  } else if (typeRoll < cleanMax + badBase) {
    type = "bad"
  } else if (typeRoll < cleanMax + badBase + bonusBase) {
    type = "bonus"
  } else {
    type = "rainbow"
  }

  drop.dataset.type = type
  drop.dataset.y = "0"
  drop.dataset.speed = (2 + Math.random() * 2).toString()
  drop.classList.add(type)

  const x = Math.random() * (gameArea.clientWidth - DROP_SIZE)
  drop.style.left = `${x}px`
  drop.style.top = "-40px"

  drop.addEventListener("click", () => {
    if (!gameRunning) return
    collectDrop(drop)
  })

  gameArea.appendChild(drop)
  drops.push(drop)
}

function updateDrops(delta) {
  const bucketRect = bucket.getBoundingClientRect()

  drops = drops.filter((drop) => {
    const y = parseFloat(drop.dataset.y) + (parseFloat(drop.dataset.speed) * delta) / 16
    drop.dataset.y = y
    drop.style.top = `${y}px`

    const dropRect = drop.getBoundingClientRect()

    if (y > gameArea.clientHeight - DROP_SIZE - 12) {
      const caught = isOverlap(dropRect, bucketRect)

      if (caught) {
        collectDrop(drop)
        return false
      }

      if (drop.dataset.type === "clean") {
        cleanMissed = true
        score = Math.max(0, score - 5)
        updateScoreDisplay()
        showFloatingText(dropRect.left + dropRect.width / 2, dropRect.top + dropRect.height / 2, "-5", "#ff5a5a")
        playSound("miss")
      }

      drop.remove()
      return false
    }

    return true
  })
}

function collectDrop(drop) {
  const type = drop.dataset.type
  const dropRect = drop.getBoundingClientRect()
  const centerX = dropRect.left + dropRect.width / 2
  const centerY = dropRect.top + dropRect.height / 2

  if (type === "clean") {
    score += 10
    cleanCaught += 1
    showFloatingText(centerX, centerY, "+10", "#3fe7ff")
    splashAt(centerX, centerY, "#6dd0ff")
    playSound("catch")

    if (cleanCaught === 1) {
      unlockAchievement("firstCatch")
    }
  } else if (type === "bad") {
    score = Math.max(0, score - 8)
    showFloatingText(centerX, centerY, "-8", "#ff5a5a")
    playSound("bad")
  } else if (type === "bonus") {
    score += 25
    showFloatingText(centerX, centerY, "+25", "#ffdf5a")
    playSound("bonus")
  } else if (type === "rainbow") {
    score += 50
    rainbowCount += 1
    timeLeft += 3
    showFloatingText(centerX, centerY, "+50!\n+3s", "#8cffd4")
    splashAt(centerX, centerY, "#a3ffdf")
    playSound("bonus")

    if (rainbowCount === 5) {
      unlockAchievement("rainbowHunter")
    }
  }

  updateScoreDisplay()
  drop.remove()
}

function isOverlap(rect1, rect2) {
  return (
    rect1.left < rect2.right &&
    rect1.right > rect2.left &&
    rect1.top < rect2.bottom &&
    rect1.bottom > rect2.top
  )
}

function moveBucketTo(x) {
  const halfWidth = bucket.clientWidth / 2
  const minX = halfWidth
  const maxX = gameArea.clientWidth - halfWidth
  const target = Math.min(maxX, Math.max(minX, x))
  bucket.style.left = `${target}px`
}

function updateScoreDisplay() {
  scoreDisplay.textContent = score
  updateMilestoneDisplay()

  if (score > highScore) {
    highScore = score
    highScoreDisplay.textContent = highScore
    localStorage.setItem("rain-rescue-highscore", highScore)
    unlockAchievement("highScore")
  }

  if (score >= nextLevelScore) {
    levelUp()
  }
}

function updateMilestoneDisplay() {
  const milestone = document.getElementById("milestone")
  if (!milestone) return

  const nextGoal = Math.ceil((score + 1) / 150) * 150
  const pointsToNext = Math.max(0, nextGoal - score)

  if (score === 0) {
    milestone.textContent = "Hit 50 points for your first milestone!"
  } else {
    milestone.textContent = `Current milestone: ${score} points, ${pointsToNext} points to next (${nextGoal}).`
  }

  if (score >= 50) {
    unlockAchievement("firstCatch")
  }
}


function levelUp() {
  level += 1
  nextLevelScore += 150
  levelDisplay.textContent = level

  if (level === 3) {
    unlockAchievement("level3")
  }

  // Speed things up and make the game more intense
  spawnInterval = Math.max(250, spawnInterval - 130)
  playSound("win")
  confetti()
  showFloatingText(gameArea.clientWidth / 2, gameArea.clientHeight / 2, `LEVEL ${level}!`, "#ffffff")
}

function endGame() {
  gameRunning = false
  cancelAnimationFrame(animationId)

  showOverlay(
    score >= 150 ? "🏆 Water Hero!" : "Game Over",
    score >= 150
      ? `Amazing! You collected ${score} points.`
      : `Try again! Your score is ${score}.`
  )

  if (score >= 150) {
    confetti()
  }

  if (!cleanMissed && score > 0) {
    unlockAchievement("noMiss")
  }

  playSound(score >= 150 ? "win" : "lose")
}

function confetti() {
  const colors = ["#ff4ec2", "#f9f871", "#4de6ff", "#68ff9d", "#ff8e72"]
  const count = 80

  for (let i = 0; i < count; i++) {
    const piece = document.createElement("div")
    piece.style.position = "fixed"
    piece.style.width = `${Math.random() * 8 + 6}px`
    piece.style.height = `${Math.random() * 14 + 6}px`
    piece.style.background = colors[Math.floor(Math.random() * colors.length)]
    piece.style.left = `${Math.random() * 100}%`
    piece.style.top = `-${Math.random() * 40 + 10}px`
    piece.style.zIndex = 9999
    piece.style.opacity = "0.9"
    piece.style.borderRadius = "3px"
    piece.style.transform = `rotate(${Math.random() * 360}deg)`
    document.body.appendChild(piece)

    const fall = setInterval(() => {
      const currentTop = parseFloat(piece.style.top)
      piece.style.top = `${currentTop + 6 + Math.random() * 4}px`
      piece.style.left = `${parseFloat(piece.style.left) + Math.random() * 4 - 2}%`

      if (currentTop > window.innerHeight) {
        piece.remove()
        clearInterval(fall)
      }
    }, 16)

    setTimeout(() => {
      piece.style.opacity = "0"
      setTimeout(() => piece.remove(), 400)
    }, 900)
  }
}

function resetGame() {
  gameRunning = false
  cancelAnimationFrame(animationId)

  score = 0
  timeLeft = GAME_DURATION
  cleanCaught = 0
  cleanMissed = false
  rainbowCount = 0

  scoreDisplay.textContent = score
  timerDisplay.textContent = timeLeft

  drops.forEach((drop) => drop.remove())
  drops = []
  showOverlay("Ready?", "Click play to start the next round.")
}

function showFloatingText(x, y, text, color) {
  const rect = gameArea.getBoundingClientRect()
  const label = document.createElement("div")
  label.className = "floating-text"
  label.textContent = text
  label.style.left = `${x - rect.left}px`
  label.style.top = `${y - rect.top}px`
  label.style.color = color
  gameArea.appendChild(label)

  setTimeout(() => {
    label.remove()
  }, 900)
}

function splashAt(x, y, color) {
  const rect = gameArea.getBoundingClientRect()
  const count = 9
  for (let i = 0; i < count; i++) {
    const dot = document.createElement("div")
    dot.className = "splash-dot"
    dot.style.background = color
    dot.style.left = `${x - rect.left}px`
    dot.style.top = `${y - rect.top}px`

    gameArea.appendChild(dot)

    const angle = (Math.PI * 2 * i) / count + (Math.random() - 0.5) * 0.5
    const speed = 1.4 + Math.random() * 1.4
    const distance = 30 + Math.random() * 18
    const dx = Math.cos(angle) * speed
    const dy = Math.sin(angle) * speed

    let progress = 0
    const step = () => {
      progress += 1
      const px = parseFloat(dot.style.left) + dx
      const py = parseFloat(dot.style.top) + dy
      dot.style.left = `${px}px`
      dot.style.top = `${py}px`
      dot.style.opacity = `${1 - progress / distance}`

      if (progress < distance) {
        requestAnimationFrame(step)
      } else {
        dot.remove()
      }
    }
    requestAnimationFrame(step)
  }
}

function showOverlay(title, text) {
  overlayTitle.textContent = title
  overlayText.textContent = text
  overlay.classList.remove("hidden")
  overlay.setAttribute("aria-hidden", "false")
}

function hideOverlay() {
  overlay.classList.add("hidden")
  overlay.setAttribute("aria-hidden", "true")
}

function playSound(type) {
  if (!soundEnabled) return
  if (!window.AudioContext && !window.webkitAudioContext) return

  const ctx = new (window.AudioContext || window.webkitAudioContext)()
  const osc = ctx.createOscillator()
  const gain = ctx.createGain()

  osc.connect(gain)
  gain.connect(ctx.destination)

  switch (type) {
    case "catch":
      osc.frequency.value = 560
      gain.gain.value = 0.12
      break
    case "bad":
      osc.frequency.value = 220
      gain.gain.value = 0.16
      break
    case "bonus":
      osc.frequency.value = 780
      gain.gain.value = 0.16
      break
    case "miss":
      osc.frequency.value = 180
      gain.gain.value = 0.1
      break
    case "start":
      osc.frequency.value = 440
      gain.gain.value = 0.14
      break
    case "win":
      osc.frequency.value = 880
      gain.gain.value = 0.18
      break
    case "lose":
      osc.frequency.value = 150
      gain.gain.value = 0.16
      break
    default:
      osc.frequency.value = 440
      gain.gain.value = 0.12
  }

  osc.type = "sine"
  osc.start()
  osc.stop(ctx.currentTime + 0.08)
}

// Show the initial overlay when the page loads
showOverlay(
  "Ready?",
  "Catch clean drops, avoid dirty ones, and grab rainbow drops for bonus time."
)
 
