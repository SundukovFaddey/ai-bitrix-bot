const express = require('express');
const axios = require('axios');
const cors = require('cors');
require('dotenv').config();

const app = express();
app.use(cors());
app.use(express.json());

const DEEPSEEK_API_KEY = process.env.DEEPSEEK_API_KEY;
const BITRIX_WEBHOOK = process.env.BITRIX_WEBHOOK;

// Хранилище сессий (в памяти, для простоты)
const sessions = new Map();

// ============= ОСНОВНЫЕ ЭНДПОИНТЫ =============

// Health check
app.get('/', (req, res) => {
    res.json({ 
        status: 'ok', 
        service: 'AI Bitrix Processor',
        version: '1.0.0',
        timestamp: new Date().toISOString()
    });
});

// Веб-хук для Битрикс24 (основной)
app.post('/webhook', async (req, res) => {
    try {
        console.log('Получен запрос от Битрикс24');
        console.log('Тело:', req.body);
        
        const { event, data } = req.body;
        
        if (event === 'ONCRMDEALADD') {
            const dealId = data.FIELDS.ID;
            const dealTitle = data.FIELDS.TITLE;
            
            console.log(`Новая сделка ${dealId}: ${dealTitle}`);
            
            // Отправляем в AI
            const aiResponse = await processWithAI(dealTitle, dealId);
            
            // Отправляем результат обратно в Битрикс24
            await sendToBitrix(dealId, aiResponse);
        }
        
        res.json({ ok: true });
    } catch (error) {
        console.error('Ошибка:', error);
        res.status(500).json({ error: error.message });
    }
});

// Прямой API для тестирования
app.post('/api/process', async (req, res) => {
    try {
        const { text, orderId } = req.body;
        const result = await processWithAI(text, orderId);
        res.json(result);
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});

// ============= ЛОГИКА ОБРАБОТКИ =============

async function processWithAI(orderText, orderId) {
    // Получаем или создаем сессию
    let session = sessions.get(orderId);
    if (!session) {
        session = {
            orderId: orderId,
            history: [],
            status: 'new'
        };
        sessions.set(orderId, session);
    }
    
    // Добавляем сообщение в историю
    session.history.push({ role: 'user', content: orderText });
    
    // Отправляем запрос в DeepSeek
    const aiMessage = await callDeepSeek(session.history);
    
    // Проверяем, нужны ли уточнения
    const needsMoreInfo = aiMessage.includes('?') || 
                          aiMessage.includes('уточните') ||
                          aiMessage.includes('скажите');
    
    if (needsMoreInfo) {
        session.history.push({ role: 'assistant', content: aiMessage });
        return {
            action: 'ask',
            question: aiMessage,
            sessionId: orderId
        };
    } else {
        // Разбиваем на подзадачи
        const subtasks = parseSubtasks(aiMessage);
        return {
            action: 'distribute',
            orderData: extractOrderData(orderText),
            subtasks: subtasks,
            sessionId: orderId
        };
    }
}

async function callDeepSeek(history) {
    const systemPrompt = `Ты - ассистент компании по производству этикеток.
    
Обязательные данные для заказа:
1. ТИРАЖ (количество штук)
2. РАЗМЕРЫ (ширина x высота в мм)
3. МАТЕРИАЛ (бумага/пленка)
4. ДИЗАЙН (есть готовый / нужна разработка)

Если данных не хватает - задай конкретный вопрос.
Если все данные есть - напиши "ГОТОВО" и перечисли их.`;
try {
        const response = await axios.post('https://api.deepseek.com/v1/chat/completions', {
            model: "deepseek-chat",
            messages: [
                { role: "system", content: systemPrompt },
                ...history
            ],
            temperature: 0.3,
            max_tokens: 1000
        }, {
            headers: {
                'Authorization': Bearer ${DEEPSEEK_API_KEY},
                'Content-Type': 'application/json'
            }
        });
        
        return response.data.choices[0].message.content;
    } catch (error) {
        console.error('DeepSeek ошибка:', error.response?.data || error.message);
        return "Извините, произошла ошибка. Попробуйте еще раз.";
    }
}

function parseSubtasks(aiResponse) {
    // Базовая структура подзадач
    return {
        technologist: {
            task: "Подобрать материалы и технологию печати",
            status: "pending"
        },
        economist: {
            task: "Рассчитать себестоимость заказа",
            status: "pending"
        },
        designer: {
            task: "Проверить/разработать макет",
            status: "pending"
        },
        production: {
            task: "Запланировать производство",
            status: "pending"
        },
        logistics: {
            task: "Рассчитать сроки и стоимость доставки",
            status: "pending"
        }
    };
}

function extractOrderData(text) {
    const data = {};
    
    const quantityMatch = text.match(/(\d+)\s*(шт|штук|тысяч)/i);
    if (quantityMatch) data.quantity = quantityMatch[1];
    
    const sizeMatch = text.match(/(\d+)[xх](\d+)/i);
    if (sizeMatch) data.dimensions = ${sizeMatch[1]}x${sizeMatch[2]} мм;
    
    return data;
}

async function sendToBitrix(dealId, aiResponse) {
    if (!BITRIX_WEBHOOK) return;
    
    try {
        if (aiResponse.action === 'ask') {
            // Добавляем вопрос в комментарий сделки
            await axios.post(`${BITRIX_WEBHOOK}crm.timeline.comment.add.json`, {
                fields: {
                    ENTITY_ID: dealId,
                    ENTITY_TYPE: 'deal',
                    COMMENT: 🤖 AI вопрос: ${aiResponse.question}
                }
            });
        } else if (aiResponse.action === 'distribute') {
            // Обновляем сделку
            await axios.post(`${BITRIX_WEBHOOK}crm.deal.update.json`, {
                id: dealId,
                fields: {
                    UF_CRM_ORDER_STATUS: 'Распределен по отделам',
                    UF_CRM_ORDER_DATA: JSON.stringify(aiResponse.orderData)
                }
            });
            
            // Добавляем комментарий
            await axios.post(`${BITRIX_WEBHOOK}crm.timeline.comment.add.json`, {
                fields: {
                    ENTITY_ID: dealId,
                    ENTITY_TYPE: 'deal',
                    COMMENT: ✅ Заказ обработан AI и распределен по отделам\n\nДанные заказа:\n${JSON.stringify(aiResponse.orderData, null, 2)}
                }
            });
        }
    } catch (error) {
        console.error('Ошибка отправки в Битрикс:', error.message);
    }
}

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
    console.log(`✅ Сервер запущен на порту ${PORT}`);
    console.log(`🌐 Веб-хук URL: https://your-service.onrender.com/webhook`);
});
