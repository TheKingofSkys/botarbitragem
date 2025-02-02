# botarbitragem
import os
import ccxt
import httpx
import asyncio
import logging
import time
from telegram import Bot

# Configurar logs
logging.basicConfig(format='%(asctime)s - %(levelname)s - %(message)s', level=logging.INFO)

# Ajustar o diretório de trabalho
diretorio_correto = "C:\\Users\\marle\\Desktop\\arbitragem_bot"  # Substitua pelo caminho correto
os.chdir(diretorio_correto)
print("Diretório alterado para:", os.getcwd())

# Configuração das exchanges suportadas pelo CCXT
exchanges = {
    'binance': {
        'exchange': ccxt.binance(),
        'url_template': 'https://www.binance.com/en/trade/{symbol}'
    },
    'coinbase': {
        'exchange': ccxt.coinbase(),
        'url_template': 'https://pro.coinbase.com/trade/{symbol}'
    },
    'kraken': {
        'exchange': ccxt.kraken(),
        'url_template': 'https://trade.kraken.com/markets/kraken/{symbol}'
    },
    'huobi': {
        'exchange': ccxt.huobi(),
        'url_template': 'https://www.huobi.com/en-us/exchange/{symbol}'
    },
    'okx': {
        'exchange': ccxt.okx(),
        'url_template': 'https://www.okx.com/trade-spot/{symbol}'
    },
    'bitfinex': {
        'exchange': ccxt.bitfinex(),
        'url_template': 'https://trading.bitfinex.com/t/{symbol}'
    },
    'bybit': {
        'exchange': ccxt.bybit(),
        'url_template': 'https://www.bybit.com/trade/spot/{symbol}'
    },
    'kucoin': {
        'exchange': ccxt.kucoin(),
        'url_template': 'https://www.kucoin.com/trade/{symbol}'
    },
    'gateio': {
        'exchange': ccxt.gateio(),
        'url_template': 'https://www.gate.io/trade/{symbol}'
    },
    'bitstamp': {
        'exchange': ccxt.bitstamp(),
        'url_template': 'https://www.bitstamp.net/markets/{symbol}/'
    },
    'novadax': {
        'exchange': ccxt.novadax(),
        'url_template': 'https://www.novadax.com.br/trade/{symbol}'
    }
}

# Configuração do Mercado Bitcoin
MERCADO_BITCOIN_API_URL_TEMPLATE = "https://www.mercadobitcoin.net/api/{symbol}/ticker/"  # Template para diferentes pares

# Configuração do bot do Telegram
TELEGRAM_BOT_TOKEN = '7863327161:AAHQgBzcTRX2cyp7jyN1un_g8Oz2tDLjef4'  # Substitua pelo token do seu bot
TELEGRAM_CHAT_ID = 829191387  # Substitua pelo Chat ID (sem aspas se for um número)
bot = Bot(token=TELEGRAM_BOT_TOKEN)

# Variáveis de controle
LIMITE_PERCENTUAL = 2.0  # Limite mínimo de spread em percentual (ex.: 2%)
INTERVALO_VERIFICACAO = 300  # Intervalo de verificação em segundos (ex.: 120 = 2 minutos)

# Cache para a taxa de conversão BRL/USDT
brl_to_usdt_rate_cache = None
brl_to_usdt_rate_timestamp = 0
CACHE_EXPIRATION = 300  # Tempo de expiração do cache em segundos (5 minutos)

# Lista de pares de moedas a serem monitorados (será preenchida dinamicamente)
PARES_MOEDAS = []

# Função para buscar a taxa de conversão USDT/BRL com cache
async def fetch_usdt_to_brl_conversion_rate():
    """
    Busca o preço de conversão USDT/BRL na NovaDAX com cache.
    """
    global brl_to_usdt_rate_cache, brl_to_usdt_rate_timestamp
    current_time = time.time()
    if brl_to_usdt_rate_cache and (current_time - brl_to_usdt_rate_timestamp < CACHE_EXPIRATION):
        logging.info("Usando taxa de conversão BRL/USDT do cache.")
        return brl_to_usdt_rate_cache
    try:
        logging.info("Buscando nova taxa de conversão USDT/BRL...")
        novadax = ccxt.novadax()
        ticker = novadax.fetch_ticker('USDT/BRL')
        usdt_brl_price = ticker['last']
        conversion_rate = 1 / usdt_brl_price  # Taxa de conversão BRL/USDT
        brl_to_usdt_rate_cache = conversion_rate
        brl_to_usdt_rate_timestamp = current_time
        logging.info(f"Nova taxa de conversão BRL/USDT obtida: {conversion_rate}")
        return conversion_rate
    except Exception as e:
        logging.error(f"Erro ao buscar preço de conversão USDT/BRL: {e}")
        return None

# Função para listar todos os pares suportados pelas exchanges
def listar_todos_pares():
    """
    Lista todos os pares suportados pelas exchanges configuradas.
    Retorna uma lista ordenada de pares únicos.
    """
    todos_pares = set()  # Usamos um conjunto para evitar duplicatas
    for nome_exchange, exchange_data in exchanges.items():
        try:
            logging.info(f"Carregando mercados da exchange: {nome_exchange}")
            markets = exchange_data['exchange'].load_markets()  # Carrega os mercados disponíveis
            pares = list(markets.keys())  # Obtém a lista de pares
            
            # Filtrar apenas pares em USDT
            pares_usdt = [par for par in pares if par.endswith('/USDT')]
            todos_pares.update(pares_usdt)  # Adiciona os pares ao conjunto
            logging.info(f"Pares em USDT encontrados na {nome_exchange}: {len(pares_usdt)}")
        except Exception as e:
            logging.error(f"Erro ao carregar mercados da {nome_exchange}: {e}")
    # Ordenar os pares alfabeticamente
    todos_pares_ordenados = sorted(todos_pares)
    return todos_pares_ordenados

# Função para verificar redes compatíveis para saques e depósitos
def verificar_redes_compativeis():
    """
    Verifica as redes suportadas para saques e depósitos em todas as exchanges.
    Retorna um dicionário com as redes compatíveis para cada ativo em cada exchange.
    """
    redes_por_exchange = {}
    for nome_exchange, exchange_data in exchanges.items():
        try:
            logging.info(f"Carregando informações de redes da exchange: {nome_exchange}")
            currencies = exchange_data['exchange'].fetch_currencies()  # Carrega informações sobre moedas e redes
            
            for currency, info in currencies.items():
                if 'networks' in info:
                    for network, network_info in info['networks'].items():
                        if network_info.get('deposit', False) and network_info.get('withdraw', False):
                            # Identificador único: moeda + rede
                            moeda_rede = f"{currency}_{network}"
                            if nome_exchange not in redes_por_exchange:
                                redes_por_exchange[nome_exchange] = {}
                            redes_por_exchange[nome_exchange][moeda_rede] = True
        except Exception as e:
            logging.error(f"Erro ao carregar informações de redes da {nome_exchange}: {e}")
    return redes_por_exchange

# Função para verificar se os saques estão habilitados para uma moeda em uma exchange
def verificar_saques_habilitados():
    """
    Verifica se os saques estão habilitados para cada moeda em cada exchange.
    Retorna um dicionário com as moedas e suas condições de saque.
    """
    saques_habilitados_por_exchange = {}
    for nome_exchange, exchange_data in exchanges.items():
        try:
            logging.info(f"Verificando status de saques na exchange: {nome_exchange}")
            currencies = exchange_data['exchange'].fetch_currencies()  # Carrega informações sobre moedas
            
            for currency, info in currencies.items():
                if info.get('withdraw', False):  # Verifica se os saques estão habilitados
                    if nome_exchange not in saques_habilitados_por_exchange:
                        saques_habilitados_por_exchange[nome_exchange] = set()
                    saques_habilitados_por_exchange[nome_exchange].add(currency)
        except Exception as e:
            logging.error(f"Erro ao verificar status de saques na {nome_exchange}: {e}")
    return saques_habilitados_por_exchange

# Função para filtrar pares com base nas redes compatíveis e saques habilitados
def filtrar_pares_por_rede_e_saques(todos_pares, redes_por_exchange, saques_habilitados_por_exchange):
    """
    Filtra os pares de trading para incluir apenas aqueles onde:
    - As redes de saque e depósito coincidem nas mesmas exchanges.
    - Os saques estão habilitados para ambas as moedas do par.
    """
    pares_filtrados = []
    for par in todos_pares:
        base_currency, quote_currency = par.split('/')  # Separa o par (ex.: BTC/USDT -> BTC, USDT)

        # Ignorar pares que não são em USDT
        if quote_currency != 'USDT':
            logging.warning(f"Par {par} ignorado porque não está cotado em USDT.")
            continue

        # Verificar se ambas as moedas têm redes compatíveis e saques habilitados nas mesmas exchanges
        exchanges_compatíveis = []
        for nome_exchange, redes_moedas in redes_por_exchange.items():
            saques_habilitados = saques_habilitados_por_exchange.get(nome_exchange, set())

            # Verificar todas as combinações possíveis de redes para as moedas do par
            for base_network in [f"{base_currency}_{net}" for net in ['ERC20', 'TRC20', 'BEP20']]:  # Exemplos de redes
                for quote_network in [f"{quote_currency}_{net}" for net in ['ERC20', 'TRC20', 'BEP20']]:
                    if (base_network in redes_moedas and quote_network in redes_moedas and
                        base_currency in saques_habilitados and quote_currency in saques_habilitados):
                        exchanges_compatíveis.append(nome_exchange)
                        logging.info(f"Par {par} incluído na exchange {nome_exchange}. Redes: {base_network}, {quote_network}")

        if exchanges_compatíveis:
            pares_filtrados.append(par)
            logging.info(f"Par {par} incluído. Exchanges compatíveis: {exchanges_compatíveis}")
        else:
            logging.warning(f"Par {par} excluído. Não há redes comuns ou saques habilitados entre {base_currency} e {quote_currency} em nenhuma exchange.")

    return pares_filtrados

# Função para verificar oportunidades de arbitragem
def check_arbitrage(prices, symbol):
    """
    Verifica se há oportunidades de arbitragem entre as exchanges.
    Retorna os detalhes da arbitragem se o spread atender ao limite percentual.
    """
    if len(prices) < 2:
        logging.warning(f"Não há preços suficientes para calcular o spread para {symbol}.")
        return None
    
    # Filtrar preços válidos (evitar valores nulos ou extremamente discrepantes)
    valid_prices = {
        name: data for name, data in prices.items()
        if data['bid'] is not None and data['ask'] is not None and 0 < data['bid'] < 1e6 and 0 < data['ask'] < 1e6
    }
    
    if len(valid_prices) < 2:
        logging.warning(f"Não há preços válidos suficientes para calcular o spread para {symbol}.")
        return None
    
    # Encontrar o maior bid (melhor preço de compra) e o menor ask (melhor preço de venda)
    max_bid_exchange = max(
        ((name, data['bid']) for name, data in valid_prices.items()),
        key=lambda x: x[1],
        default=(None, None)
    )
    min_ask_exchange = min(
        ((name, data['ask']) for name, data in valid_prices.items()),
        key=lambda x: x[1],
        default=(None, None)
    )
    
    if max_bid_exchange[1] is None or min_ask_exchange[1] is None:
        logging.warning(f"Preços inválidos para calcular o spread para {symbol}.")
        return None
    
    max_bid_exchange_name, max_bid = max_bid_exchange
    min_ask_exchange_name, min_ask = min_ask_exchange
    
    spread = max_bid - min_ask
    if spread <= 0:  # Ignorar spreads negativos ou inválidos
        logging.warning(f"Spread inválido detectado para {symbol}: ${spread:.2f}")
        return None
    
    percentual_spread = (spread / min_ask) * 100  # Calcula o spread em percentual
    
    logging.info(f"Spread calculado para {symbol}: ${spread:.2f} ({percentual_spread:.2f}%)")
    
    # Limitar o spread máximo a 20%
    if percentual_spread > 20:
        logging.warning(f"Spread muito alto para {symbol}: {percentual_spread:.2f}%. Ignorando.")
        return None
    
    if percentual_spread >= LIMITE_PERCENTUAL:
        return {
            'symbol': symbol,
            'min_ask_exchange': min_ask_exchange_name,
            'max_bid_exchange': max_bid_exchange_name,
            'min_ask': min_ask,
            'max_bid': max_bid,
            'spread': spread,
            'percentual_spread': percentual_spread
        }
    return None

# Função para enviar mensagens para o Telegram
async def send_telegram_message(message):
    """
    Envia uma mensagem para o Telegram.
    """
    try:
        logging.info("Enviando mensagem...")
        await bot.send_message(chat_id=TELEGRAM_CHAT_ID, text=message, disable_web_page_preview=True)
        logging.info("Mensagem enviada com sucesso!")
    except Exception as e:
        logging.error(f"Erro ao enviar mensagem: {e}")

# Função para buscar preços nas exchanges
async def fetch_prices(symbol):
    """
    Busca os preços de compra (bid) e venda (ask) do par de trading em diferentes exchanges.
    """
    prices = {}
    brl_to_usdt_rate = await fetch_usdt_to_brl_conversion_rate()  # Obter taxa de conversão BRL/USDT
    
    if not brl_to_usdt_rate:
        logging.warning("Não foi possível obter a taxa de conversão BRL/USDT. Usando apenas exchanges em USDT.")
    
    async def fetch_exchange_price(name, exchange_data):
        try:
            logging.info(f"Buscando preço na exchange: {name}")
            ticker = exchange_data['exchange'].fetch_ticker(symbol)
            bid = ticker.get('bid')  # Melhor preço de compra
            ask = ticker.get('ask')  # Melhor preço de venda
            
            # Validar preços para evitar valores absurdos
            if bid is None or ask is None or bid <= 0 or ask <= 0 or bid > 1e6 or ask > 1e6:
                logging.warning(f"Preço inválido na {name}: bid={bid}, ask={ask}")
                return name, None, None
            
            return name, bid, ask
        except Exception as e:
            logging.error(f"Erro ao buscar preço na {name}: {e}")
            return name, None, None
    
    # Buscar preços de todas as exchanges em paralelo
    tasks = [fetch_exchange_price(name, data) for name, data in exchanges.items()]
    results = await asyncio.gather(*tasks)
    
    # Filtrar resultados válidos
    for name, bid, ask in results:
        if bid and ask:
            prices[name] = {'bid': bid, 'ask': ask}
            logging.info(f"Preços obtidos na {name}: bid={bid}, ask={ask}")
    
    # Adiciona o Mercado Bitcoin
    try:
        mercado_bitcoin_api_url = MERCADO_BITCOIN_API_URL_TEMPLATE.format(symbol=symbol.split('/')[0])
        logging.info(f"Buscando preço no Mercado Bitcoin para o par {symbol}...")
        async with httpx.AsyncClient(timeout=10.0) as client:
            response = await client.get(mercado_bitcoin_api_url)
            if response.status_code == 200:
                data = response.json()
                mercado_bitcoin_price_brl = float(data['ticker']['buy'])  # Preço de compra (bid) em BRL
                if brl_to_usdt_rate:
                    mercado_bitcoin_price_usdt = mercado_bitcoin_price_brl * brl_to_usdt_rate  # Converter para USDT
                    
                    # Validar preço convertido
                    if mercado_bitcoin_price_usdt > 0 and mercado_bitcoin_price_usdt < 1e6:
                        prices['mercadobitcoin'] = {'bid': mercado_bitcoin_price_usdt, 'ask': None}
                        logging.info(f"Preço obtido no Mercado Bitcoin (convertido para USDT): {mercado_bitcoin_price_usdt}")
                    else:
                        logging.warning("Preço convertido do Mercado Bitcoin inválido.")
                else:
                    logging.warning("Não foi possível converter o preço do Mercado Bitcoin para USDT.")
            else:
                logging.error(f"Erro ao buscar preço no Mercado Bitcoin para o par {symbol}: Status Code {response.status_code}")
    except Exception as e:
        logging.error(f"Erro ao buscar preço no Mercado Bitcoin para o par {symbol}: {e}")
    
    return prices

# Função principal para verificar oportunidades de arbitragem periodicamente
async def verificar_periodicamente():
    """
    Função que verifica oportunidades de arbitragem periodicamente para múltiplos pares.
    """
    global INTERVALO_VERIFICACAO
    
    while True:  # Loop principal para manter o bot rodando continuamente
        for symbol in PARES_MOEDAS:  # Iterar sobre todos os pares
            formatted_symbol = symbol.replace('/', '_')  # Formatar o símbolo para URLs (ex.: BTC_USDT)
            
            logging.info(f"Verificando oportunidades de arbitragem para o par {symbol}...")
            prices = await fetch_prices(symbol)
            
            if not prices:
                logging.warning(f"Nenhum preço encontrado nas exchanges para o par {symbol}.")
                continue
            
            logging.info(f"Preços obtidos para {symbol}: {prices}")
            arbitrage_opportunity = check_arbitrage(prices, symbol)
            
            if arbitrage_opportunity:
                min_ask_exchange = arbitrage_opportunity['min_ask_exchange']
                max_bid_exchange = arbitrage_opportunity['max_bid_exchange']
                
                # Gerar links para as exchanges
                min_ask_url = exchanges.get(min_ask_exchange, {}).get('url_template', '').format(symbol=formatted_symbol)
                max_bid_url = exchanges.get(max_bid_exchange, {}).get('url_template', '').format(symbol=formatted_symbol)
                
                # Tratamento especial para Mercado Bitcoin
                if min_ask_exchange == 'mercadobitcoin':
                    min_ask_url = f"https://www.mercadobitcoin.com.br/trade/{symbol.split('/')[0]}BRL"
                if max_bid_exchange == 'mercadobitcoin':
                    max_bid_url = f"https://www.mercadobitcoin.com.br/trade/{symbol.split('/')[0]}BRL"
                
                message = (
                    f"🚨 Oportunidade de Arbitragem 🚀\n"
                    f"Ativo: {arbitrage_opportunity['symbol']}\n"
                    f"Compra em: [{min_ask_exchange}]({min_ask_url}) (${arbitrage_opportunity['min_ask']:.2f})\n"
                    f"Venda em: [{max_bid_exchange}]({max_bid_url}) (${arbitrage_opportunity['max_bid']:.2f})\n"
                    f"Spread: ${arbitrage_opportunity['spread']:.2f} "
                    f"({arbitrage_opportunity['percentual_spread']:.2f}%)\n"
                    f"Limite mínimo: {LIMITE_PERCENTUAL}%"
                )
                await send_telegram_message(message)
            else:
                logging.info(f"Nenhuma oportunidade de arbitragem encontrada para o par {symbol} ou spread abaixo do limite.")
        
        # Aguarda o intervalo antes de verificar novamente
        logging.info(f"Aguardando {INTERVALO_VERIFICACAO} segundos antes da próxima verificação...")
        await asyncio.sleep(INTERVALO_VERIFICACAO)

# Bloco principal atualizado
if __name__ == "__main__":
    try:
        # Listar todos os pares suportados pelas exchanges
        logging.info("Listando todos os pares suportados pelas exchanges...")
        todos_pares = listar_todos_pares()

        # Verificar redes compatíveis para saques e depósitos
        logging.info("Verificando redes compatíveis para saques e depósitos...")
        redes_por_exchange = verificar_redes_compativeis()

        # Verificar status de saques para cada moeda
        logging.info("Verificando status de saques para cada moeda...")
        saques_habilitados_por_exchange = verificar_saques_habilitados()

        # Filtrar pares com base nas redes compatíveis e saques habilitados
        logging.info("Filtrando pares com base nas redes compatíveis e saques habilitados...")
        PARES_MOEDAS = filtrar_pares_por_rede_e_saques(todos_pares, redes_por_exchange, saques_habilitados_por_exchange)
        logging.info(f"Pares selecionados para monitoramento: {PARES_MOEDAS}")

        # Executar a função assíncrona principal
        asyncio.run(verificar_periodicamente())
    except KeyboardInterrupt:
        logging.info("\nLoop interrompido pelo usuário. Encerrando o bot...")
    except Exception as e:
        logging.error(f"Erro ao configurar o bot: {e}")

