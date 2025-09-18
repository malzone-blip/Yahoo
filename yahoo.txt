import yfinance as yf
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt

def analisar_acao(ticker, periodo='6mo'):
    """
    Função para analisar uma ação com base em indicadores técnicos
    """
    # Buscar dados históricos
    acao = yf.Ticker(ticker)
    dados = acao.history(period=periodo)
    
    # Calcular indicadores técnicos
    # Médias móveis
    dados['SMA_20'] = dados['Close'].rolling(window=20).mean()
    dados['SMA_50'] = dados['Close'].rolling(window=50).mean()
    
    # RSI (Relative Strength Index)
    delta = dados['Close'].diff()
    gain = (delta.where(delta > 0, 0)).rolling(window=14).mean()
    loss = (-delta.where(delta < 0, 0)).rolling(window=14).mean()
    rs = gain / loss
    dados['RSI'] = 100 - (100 / (1 + rs))
    
    # Bandas de Bollinger
    dados['BB_Middle'] = dados['Close'].rolling(window=20).mean()
    bb_std = dados['Close'].rolling(window=20).std()
    dados['BB_Upper'] = dados['BB_Middle'] + (bb_std * 2)
    dados['BB_Lower'] = dados['BB_Middle'] - (bb_std * 2)
    
    # MACD
    exp12 = dados['Close'].ewm(span=12, adjust=False).mean()
    exp26 = dados['Close'].ewm(span=26, adjust=False).mean()
    dados['MACD'] = exp12 - exp26
    dados['MACD_Signal'] = dados['MACD'].ewm(span=9, adjust=False).mean()
    dados['MACD_Histogram'] = dados['MACD'] - dados['MACD_Signal']
    
    # Análise atual
    ultimo_preco = dados['Close'].iloc[-1]
    rsi_atual = dados['RSI'].iloc[-1]
    
    # Definir preços alvo com base em análise técnica
    resistencia = dados['BB_Upper'].iloc[-1]
    suporte = dados['BB_Lower'].iloc[-1]
    
    # Sugestão de operação
    if rsi_atual < 35 and ultimo_preco < dados['SMA_20'].iloc[-1]:
        recomendacao = "COMPRA"
        preco_alvo_compra = max(suporte, ultimo_preco * 0.97)  # 3% abaixo ou no suporte
        preco_alvo_venda = resistencia * 0.98  # 2% abaixo da resistência
    elif rsi_atual > 65 and ultimo_preco > dados['SMA_20'].iloc[-1]:
        recomendacao = "VENDA"
        preco_alvo_compra = suporte * 1.02  # 2% acima do suporte
        preco_alvo_venda = min(resistencia, ultimo_preco * 1.03)  # 3% acima ou na resistência
    else:
        recomendacao = "NEUTRO"
        preco_alvo_compra = ultimo_preco * 0.95
        preco_alvo_venda = ultimo_preco * 1.08
    
    return {
        'ticker': ticker,
        'preco_atual': ultimo_preco,
        'recomendacao': recomendacao,
        'rsi': rsi_atual,
        'suporte': suporte,
        'resistencia': resistencia,
        'preco_alvo_compra': round(preco_alvo_compra, 2),
        'preco_alvo_venda': round(preco_alvo_venda, 2),
        'dados': dados
    }

# Exemplo de uso para algumas ações brasileiras
acoes = ['PETR4.SA', 'VALE3.SA', 'ITUB4.SA', 'BBDC4.SA']

for acao in acoes:
    resultado = analisar_acao(acao)
    print(f"\nAnálise para {resultado['ticker']}:")
    print(f"Preço atual: R$ {resultado['preco_atual']:.2f}")
    print(f"Recomendação: {resultado['recomendacao']}")
    print(f"RSI: {resultado['rsi']:.2f}")
    print(f"Suporte: R$ {resultado['suporte']:.2f}")
    print(f"Resistência: R$ {resultado['resistencia']:.2f}")
    print(f"Preço alvo para compra: R$ {resultado['preco_alvo_compra']}")
    print(f"Preço alvo para venda: R$ {resultado['preco_alvo_venda']}")

# Gerar gráfico para a primeira ação
resultado = analisar_acao(acoes[0])
dados = resultado['dados']

plt.figure(figsize=(12, 8))

# Gráfico de preços
plt.subplot(2, 1, 1)
plt.plot(dados.index, dados['Close'], label='Preço')
plt.plot(dados.index, dados['SMA_20'], label='Média 20 dias')
plt.plot(dados.index, dados['SMA_50'], label='Média 50 dias')
plt.plot(dados.index, dados['BB_Upper'], 'r--', label='Bollinger Superior')
plt.plot(dados.index, dados['BB_Lower'], 'g--', label='Bollinger Inferior')
plt.title(f'Análise Técnica - {acoes[0]}')
plt.legend()
plt.grid(True)

# Gráfico de RSI
plt.subplot(2, 1, 2)
plt.plot(dados.index, dados['RSI'], label='RSI')
plt.axhline(y=70, color='r', linestyle='--', label='Sobrevendido (70)')
plt.axhline(y=30, color='g', linestyle='--', label='Sobrecomprado (30)')
plt.legend()
plt.grid(True)

plt.tight_layout()
plt.show()