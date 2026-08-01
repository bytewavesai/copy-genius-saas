const express = require('express');
const stripe = require('stripe')(process.env.STRIPE_SECRET_KEY);
const cors = require('cors');
const axios = require('axios');
require('dotenv').config();

const app = express();
app.use(express.json());
app.use(cors());

// Servir arquivos estáticos (seu HTML)
app.use(express.static('public'));

/**
 * POST /api/generate-copy
 * Gera um copy com Claude API
 */
app.post('/api/generate-copy', async (req, res) => {
    try {
        const { productName, category, description, audience, tone } = req.body;

        // Validação
        if (!productName || !description || !audience) {
            return res.status(400).json({ error: 'Campos obrigatórios faltando' });
        }

        // Prompt para Claude
        const prompt = `Você é um especialista em copywriting para e-commerce. Gere um copy profissional e convincente para vender o seguinte produto:

Produto: ${productName}
Categoria: ${category}
Descrição: ${description}
Público-alvo: ${audience}
Estilo: ${tone}

Faça um copy de 150-200 palavras que:
1. Chame atenção no primeiro parágrafo
2. Destaque os principais benefícios
3. Crie urgência ou desejo de compra
4. Inclua um chamado à ação forte
5. Seja otimizado para plataformas de venda online

Retorne apenas o copy, sem explicações extras.`;

        // Chamar Claude API
        const response = await axios.post(
            'https://api.anthropic.com/v1/messages',
            {
                model: 'claude-opus-4-1',
                max_tokens: 1000,
                messages: [{ role: 'user', content: prompt }]
            },
            {
                headers: {
                    'x-api-key': process.env.CLAUDE_API_KEY,
                    'anthropic-version': '2023-06-01',
                    'Content-Type': 'application/json'
                }
            }
        );

        const copy = response.data.content[0].text;

        res.json({ 
            success: true, 
            copy: copy,
            tokensUsed: response.data.usage.output_tokens 
        });
    } catch (error) {
        console.error('Erro ao gerar copy:', error);
        res.status(500).json({ error: 'Erro ao gerar copy com IA' });
    }
});

/**
 * POST /api/create-payment-intent
 * Cria um Payment Intent no Stripe
 */
app.post('/api/create-payment-intent', async (req, res) => {
    try {
        const paymentIntent = await stripe.paymentIntents.create({
            amount: 500, // R$ 5,00 em centavos
            currency: 'brl',
            payment_method_types: ['card'],
            description: 'Copy AI Generator - 1 Copy'
        });

        res.json({ 
            clientSecret: paymentIntent.client_secret,
            intentId: paymentIntent.id
        });
    } catch (error) {
        console.error('Erro Stripe:', error);
        res.status(400).json({ error: error.message });
    }
});

/**
 * POST /api/confirm-payment
 * Confirma o pagamento
 */
app.post('/api/confirm-payment', async (req, res) => {
    try {
        const { paymentIntentId, paymentMethodId } = req.body;

        // Confirmar o payment intent
        const paymentIntent = await stripe.paymentIntents.confirm(
            paymentIntentId,
            { payment_method: paymentMethodId }
        );

        if (paymentIntent.status === 'succeeded') {
            res.json({ 
                success: true, 
                status: paymentIntent.status,
                amount: paymentIntent.amount / 100 // Converter centavos para reais
            });
        } else if (paymentIntent.status === 'requires_action') {
            res.json({ 
                success: false, 
                status: paymentIntent.status,
                clientSecret: paymentIntent.client_secret
            });
        } else {
            res.json({ 
                success: false, 
                status: paymentIntent.status 
            });
        }
    } catch (error) {
        console.error('Erro ao confirmar pagamento:', error);
        res.status(400).json({ error: error.message });
    }
});

/**
 * GET /api/payment-status/:intentId
 * Verifica o status de um pagamento
 */
app.get('/api/payment-status/:intentId', async (req, res) => {
    try {
        const paymentIntent = await stripe.paymentIntents.retrieve(req.params.intentId);
        res.json({ 
            status: paymentIntent.status,
            amount: paymentIntent.amount / 100
        });
    } catch (error) {
        res.status(400).json({ error: error.message });
    }
});

/**
 * POST /api/webhook
 * Webhook do Stripe para eventos
 */
app.post('/api/webhook', express.raw({type: 'application/json'}), async (req, res) => {
    const sig = req.headers['stripe-signature'];
    
    try {
        const event = stripe.webhooks.constructEvent(
            req.body,
            sig,
            process.env.STRIPE_WEBHOOK_SECRET
        );

        // Processar diferentes tipos de eventos
        switch (event.type) {
            case 'payment_intent.succeeded':
                console.log('Pagamento bem-sucedido:', event.data.object.id);
                // Aqui você pode salvar em banco de dados, enviar email, etc.
                break;
            case 'payment_intent.payment_failed':
                console.log('Pagamento falhou:', event.data.object.id);
                break;
        }

        res.json({ received: true });
    } catch (error) {
        console.error('Webhook error:', error);
        res.status(400).json({ error: error.message });
    }
});

/**
 * Health check
 */
app.get('/api/health', (req, res) => {
    res.json({ status: 'ok', timestamp: new Date().toISOString() });
});

// Porta
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
    console.log(`🚀 Servidor rodando na porta ${PORT}`);
    console.log(`Claude API Key: ${process.env.CLAUDE_API_KEY ? '✓ Configurada' : '✗ Não configurada'}`);
    console.log(`Stripe Secret Key: ${process.env.STRIPE_SECRET_KEY ? '✓ Configurada' : '✗ Não configurada'}`);
});
