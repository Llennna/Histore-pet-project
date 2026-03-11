<script>
import { ref, computed, onMounted } from 'vue'

export default {
  name: 'App',
  setup() {
    // Состояния
    const user = ref(null)
    const currentQuestion = ref(null)
    const answered = ref(false)
    const selectedAnswer = ref(null)
    const loading = ref(false)
    const leaderboard = ref([])
    const userAchievements = ref([])
    const leaderboardVisible = ref(false)
    const achievementsVisible = ref(false)
    const notification = ref(null)

    // ✅ Берем URL из переменной окружения
    const API_URL = import.meta.env.VITE_API_URL || 'http://localhost:8000'

    // Опции для текущего вопроса
    const options = computed(() => {
      if (!currentQuestion.value) return []
      return [
        { letter: 'A', text: currentQuestion.value.option_a },
        { letter: 'B', text: currentQuestion.value.option_b },
        { letter: 'C', text: currentQuestion.value.option_c },
        { letter: 'D', text: currentQuestion.value.option_d }
      ]
    })

    // Загрузка при монтировании
    onMounted(async () => {
      await initTelegram()
      await loadLeaderboard()
    })

    // Инициализация Telegram
    async function initTelegram() {
      try {
        if (window.Telegram?.WebApp) {
          const tg = window.Telegram.WebApp
          tg.ready()
          tg.expand()
          
          // Авторизация на бэкенде
          await authUser(tg.initDataUnsafe.user)
        } else {
          // Для теста вне Telegram
          await authUser({
            id: '123456',
            first_name: 'Тестовый',
            username: 'test_user'
          })
        }
      } catch (error) {
        showNotification('Ошибка подключения к Telegram', 'error')
        console.error(error)
      }
    }

    // Авторизация пользователя
    async function authUser(telegramUser) {
      try {
        const response = await fetch(`${API_URL}/api/users/auth`, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(telegramUser)
        })
        user.value = await response.json()
        showNotification(`Привет, ${user.value.first_name}!`, 'success')
      } catch (error) {
        showNotification('Ошибка авторизации', 'error')
        console.error(error)
      }
    }

    // Начать игру
    async function startGame() {
      leaderboardVisible.value = false
      achievementsVisible.value = false
      await loadQuestion()
    }

    // Загрузить вопрос
    async function loadQuestion() {
      loading.value = true
      answered.value = false
      selectedAnswer.value = null
      
      try {
        const response = await fetch(`${API_URL}/api/questions/random`)
        currentQuestion.value = await response.json()
      } catch (error) {
        showNotification('Ошибка загрузки вопроса', 'error')
        console.error(error)
      } finally {
        loading.value = false
      }
    }

    // Обработка ответа
    async function handleAnswer(letter) {
      if (answered.value) return
      
      answered.value = true
      selectedAnswer.value = letter
      
      const isCorrect = letter === currentQuestion.value.correct_answer
      
      if (isCorrect && user.value) {
        // Обновляем счет пользователя
        user.value.total_score = (user.value.total_score || 0) + 10
        
        // Отправляем результат на бэкенд
        try {
          await fetch(`${API_URL}/api/quiz/answer`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({
              user_id: user.value.id,
              question_id: currentQuestion.value.id,
              correct: true
            })
          })
          
          showNotification('+10 очков!', 'success')
        } catch (error) {
          console.error('Ошибка сохранения результата', error)
        }
      } else {
        showNotification('Попробуй еще раз!', 'info')
      }
    }

    // Следующий вопрос
    function nextQuestion() {
      loadQuestion()
    }

    // Показать лидерборд
    async function showLeaderboard() {
      leaderboardVisible.value = !leaderboardVisible.value
      achievementsVisible.value = false
      if (leaderboardVisible.value) {
        await loadLeaderboard()
      }
    }

    // Загрузить лидерборд
    async function loadLeaderboard() {
      try {
        const response = await fetch(`${API_URL}/api/leaderboard?limit=10`)
        leaderboard.value = await response.json()
      } catch (error) {
        console.error('Ошибка загрузки лидерборда', error)
        showNotification('Ошибка загрузки лидерборда', 'error')
      }
    }

    // Показать достижения
    async function showAchievements() {
      achievementsVisible.value = !achievementsVisible.value
      leaderboardVisible.value = false
      if (achievementsVisible.value && user.value) {
        await loadUserAchievements()
      }
    }

    // Загрузить достижения пользователя
    async function loadUserAchievements() {
      try {
        const response = await fetch(`${API_URL}/api/user/${user.value.id}/achievements`)
        userAchievements.value = await response.json()
      } catch (error) {
        console.error('Ошибка загрузки достижений', error)
        showNotification('Ошибка загрузки достижений', 'error')
      }
    }

    // Показать уведомление
    function showNotification(message, type = 'info') {
      notification.value = { message, type }
      setTimeout(() => {
        notification.value = null
      }, 3000)
    }

    // Получить текст сложности
    function getDifficultyText(difficulty) {
      const map = {
        'easy': 'Легкий',
        'medium': 'Средний',
        'hard': 'Сложный'
      }
      return map[difficulty] || difficulty
    }

    return {
      user,
      currentQuestion,
      answered,
      selectedAnswer,
      loading,
      leaderboard,
      userAchievements,
      leaderboardVisible,
      achievementsVisible,
      notification,
      options,
      startGame,
      handleAnswer,
      nextQuestion,
      showLeaderboard,
      showAchievements,
      getDifficultyText
    }
  }
}
</script>
<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, sans-serif;
  background: var(--tg-theme-bg-color, #f5f5f5);
  color: var(--tg-theme-text-color, #222);
  line-height: 1.6;
}

.app {
  max-width: 600px;
  margin: 0 auto;
  padding: 20px;
  min-height: 100vh;
  display: flex;
  flex-direction: column;
}

/* Шапка */
header {
  margin-bottom: 30px;
  text-align: center;
}

h1 {
  font-size: 1.8rem;
  margin-bottom: 15px;
  color: var(--tg-theme-button-color, #2481cc);
}

.user-info {
  display: flex;
  align-items: center;
  gap: 15px;
  padding: 15px;
  background: var(--tg-theme-secondary-bg-color, #e8e8e8);
  border-radius: 12px;
  margin-top: 10px;
}

.user-avatar {
  width: 50px;
  height: 50px;
  border-radius: 50%;
  overflow: hidden;
}

.user-avatar img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.user-details {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 5px;
}

.user-name {
  font-weight: 600;
  font-size: 1.1rem;
}

.user-score {
  font-size: 0.9rem;
  color: var(--tg-theme-hint-color, #666);
}

/* Загрузка */
.loading {
  text-align: center;
  padding: 40px;
  color: var(--tg-theme-hint-color, #666);
}

/* Главное меню */
.menu-screen {
  display: flex;
  flex-direction: column;
  gap: 20px;
}

.play-btn {
  width: 100%;
  padding: 25px;
  font-size: 1.8rem;
  background: var(--tg-theme-button-color, #40a7e3);
  color: var(--tg-theme-button-text-color, white);
  border: none;
  border-radius: 20px;
  cursor: pointer;
  transition: transform 0.2s, opacity 0.2s;
}

.play-btn:hover {
  transform: scale(1.02);
  opacity: 0.9;
}

.menu-buttons {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 10px;
  margin-bottom: 20px;
}

.menu-btn {
  padding: 15px;
  background: var(--tg-theme-secondary-bg-color, #e8e8e8);
  color: var(--tg-theme-text-color, #222);
  border: none;
  border-radius: 12px;
  font-size: 1.1rem;
  cursor: pointer;
  transition: opacity 0.2s;
}

.menu-btn:hover {
  opacity: 0.8;
}

/* Игровой экран */
.game-screen {
  display: flex;
  flex-direction: column;
  gap: 20px;
}

.question-header {
  display: flex;
  gap: 10px;
  margin-bottom: 10px;
}

.difficulty {
  padding: 5px 12px;
  border-radius: 20px;
  font-size: 0.9rem;
  font-weight: 500;
}

.difficulty.easy {
  background: #4caf50;
  color: white;
}

.difficulty.medium {
  background: #ff9800;
  color: white;
}

.difficulty.hard {
  background: #f44336;
  color: white;
}

.era {
  padding: 5px 12px;
  background: var(--tg-theme-secondary-bg-color, #e8e8e8);
  border-radius: 20px;
  font-size: 0.9rem;
}

.question {
  font-size: 1.3rem;
  margin-bottom: 20px;
  line-height: 1.5;
}

.options {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.option-btn {
  padding: 15px 20px;
  background: var(--tg-theme-button-color, #40a7e3);
  color: var(--tg-theme-button-text-color, white);
  border: none;
  border-radius: 12px;
  text-align: left;
  font-size: 1rem;
  cursor: pointer;
  transition: all 0.3s;
  position: relative;
  overflow: hidden;
}

.option-btn:not(:disabled):hover {
  transform: translateX(5px);
  opacity: 0.9;
}

.option-btn:disabled {
  opacity: 0.7;
  cursor: not-allowed;
}

.option-btn.correct {
  background: #4caf50;
  animation: pulse 0.5s;
}

.option-btn.wrong {
  background: #f44336;
  opacity: 0.5;
}

.option-letter {
  font-weight: bold;
  margin-right: 15px;
  display: inline-block;
  min-width: 25px;
}

@keyframes pulse {
  0% { transform: scale(1); }
  50% { transform: scale(1.02); }
  100% { transform: scale(1); }
}

/* Обратная связь после ответа */
.answer-feedback {
  margin-top: 20px;
  padding: 15px;
  border-radius: 12px;
  text-align: center;
}

.correct-feedback {
  color: #4caf50;
  font-weight: 600;
  margin-bottom: 10px;
  font-size: 1.2rem;
}

.wrong-feedback {
  color: #f44336;
  margin-bottom: 10px;
}

.next-btn {
  width: 100%;
  padding: 15px;
  background: var(--tg-theme-button-color, #40a7e3);
  color: var(--tg-theme-button-text-color, white);
  border: none;
  border-radius: 12px;
  font-size: 1.1rem;
  cursor: pointer;
  transition: opacity 0.2s;
}

.next-btn:hover {
  opacity: 0.9;
}

/* Лидерборд */
.leaderboard, .achievements {
  margin-top: 20px;
  padding: 20px;
  background: var(--tg-theme-secondary-bg-color, #e8e8e8);
  border-radius: 12px;
}

.leaderboard h3, .achievements h3 {
  margin-bottom: 15px;
  color: var(--tg-theme-text-color, #222);
}

.leaderboard-item {
  display: flex;
  align-items: center;
  padding: 12px;
  background: var(--tg-theme-bg-color, white);
  margin-bottom: 8px;
  border-radius: 8px;
}

.rank {
  font-weight: bold;
  min-width: 30px;
  color: var(--tg-theme-hint-color, #666);
}

.player-name {
  flex: 1;
  font-weight: 500;
}

.player-score {
  font-weight: 600;
  color: #ffc107;
}

/* Достижения */
.achievement-item {
  display: flex;
  align-items: center;
  gap: 15px;
  padding: 15px;
  background: var(--tg-theme-bg-color, white);
  margin-bottom: 8px;
  border-radius: 8px;
}

.achievement-icon {
  font-size: 2rem;
  min-width: 40px;
  text-align: center;
}

.achievement-info {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.achievement-name {
  font-weight: 600;
}

.achievement-desc {
  font-size: 0.9rem;
  color: var(--tg-theme-hint-color, #666);
}

.empty-state {
  text-align: center;
  padding: 30px;
  color: var(--tg-theme-hint-color, #666);
}

/* Уведомления */
.notification {
  position: fixed;
  bottom: 20px;
  left: 20px;
  right: 20px;
  max-width: 400px;
  margin: 0 auto;
  padding: 15px 20px;
  border-radius: 12px;
  color: white;
  font-weight: 500;
  text-align: center;
  animation: slideUp 0.3s ease;
  z-index: 1000;
}

.notification.success {
  background: #4caf50;
}

.notification.error {
  background: #f44336;
}

.notification.info {
  background: #2196f3;
}

@keyframes slideUp {
  from {
    transform: translateY(100%);
    opacity: 0;
  }
  to {
    transform: translateY(0);
    opacity: 1;
  }
}

/* Адаптивность */
@media (max-width: 480px) {
  .app {
    padding: 15px;
  }
  
  h1 {
    font-size: 1.5rem;
  }
  
  .play-btn {
    font-size: 1.5rem;
    padding: 20px;
  }
  
  .question {
    font-size: 1.1rem;
  }
  
  .menu-buttons {
    grid-template-columns: 1fr;
  }
}
</style>