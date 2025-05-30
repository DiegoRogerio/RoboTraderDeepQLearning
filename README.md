# RoboTraderDeepQLearning

{
  "cells": [
    {
      "cell_type": "markdown",
      "source": [
        "# Robo Trade 2.0 Usando Deep Q-Learning\n",
        "\n",
        "Segunda tentativa de criação de uma rede neural para prever oscilações no mercado de ação.\n",
        "\n",
        "Desta vez tentaremos usar Deep Q-Learning, usando Q-Networks.\n",
        "\n",
        "<hr>\n",
        "\n",
        "## Funcionamento\n",
        "\n",
        "Nossa rede neural terá a seguinte configuração:\n",
        "\n",
        "* **Entrada:** os últimos X estados escolhidos pela variável ```window_size```.\n",
        "* **Saídas:** Comprar, Vender, Esperar.\n",
        "\n",
        "<hr>\n",
        "\n",
        "## Classes e suas funções\n",
        "### **AI_Trader:**\n",
        "- **Construtor:**\n",
        "\n",
        "  Inicializa o agente que atuará no nosso ambiente.\n",
        "\n",
        "- **Model Builder:**\n",
        "\n",
        "  Modela a arquitetura da nossa rede neural de acordo com nossas escolhas.\n",
        "\n",
        "- **Trade:**\n",
        "\n",
        "  Função de decide se o agente irá executar um previsão usando a Q-Network ou executará uma ação gananciosa aleatória.\n",
        "\n",
        "  Esta também é a função que retorna a resposta final da rede neural (Comprar, Vender ou Esperar).\n",
        "\n",
        "- **Batch_Trade:**\n",
        "\n",
        "  Função que realiza o treinamendo do lote de memórias.\n",
        "\n",
        "\n",
        "\n"
      ],
      "metadata": {
        "id": "QSFHJ1IYFx3K"
      }
    },
    {
      "cell_type": "code",
      "source": [
        "# EXECUTAR NA PRIMEIRA EXECUÇÃO!"
      ],
      "metadata": {
        "id": "ejU_s_D7j6b6"
      },
      "execution_count": None,
      "outputs": []
    },
    {
      "cell_type": "code",
      "source": [
        "pip install yfinance"
      ],
      "metadata": {
        "id": "ukXLTyxQA4oQ"
      },
      "execution_count": None,
      "outputs": []
    },
    {
      "cell_type": "code",
      "source": [
        "# EXECUTAR NA PRIMEIRA EXECUÇÃO!\n",
        "\n",
        "# Bitcoin Hora a hora - Data download\n",
        "!gdown --id '1VQry5JMRcuZ_BStIX8-FB8n4zFuDNUjk'"
      ],
      "metadata": {
        "id": "6wn_pMN7kC2s",
        "colab": {
          "base_uri": "https://localhost:8080/"
        },
        "outputId": "0eea3dae-3513-4deb-fa85-3ee21d3b2dee"
      },
      "execution_count": None,
      "outputs": [
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            "Downloading...\n",
            "From: https://drive.google.com/uc?id=1VQry5JMRcuZ_BStIX8-FB8n4zFuDNUjk\n",
            "To: /content/BITCOIN_HORA_A_HORA_2019_2022_BEST_DATA_2.csv\n",
            "\r  0% 0.00/1.95M [00:00<?, ?B/s]\r100% 1.95M/1.95M [00:00<00:00, 168MB/s]\n"
          ]
        }
      ]
    },
    {
      "cell_type": "code",
      "source": [
        "# Imports\n",
        "\n",
        "import math\n",
        "import random\n",
        "import numpy as np\n",
        "import pandas as pd\n",
        "import tensorflow as tf\n",
        "import matplotlib.pyplot as plt\n",
        "import pandas_datareader as data_reader\n",
        "\n",
        "from tqdm import tqdm_notebook, tqdm\n",
        "from collections import deque\n",
        "\n",
        "import pandas\n",
        "from pandas_datareader import data as pdr\n",
        "import yfinance as yfin\n",
        "import datetime\n",
        "\n",
        "import numpy as np\n",
        "from tensorflow import keras\n",
        "from tensorflow.keras import layers"
      ],
      "metadata": {
        "id": "QTd21an3F2a5"
      },
      "execution_count": None,

      "outputs": []
    },
    {
      "cell_type": "code",
      "source": [
        "# Defining our Deep Q-Learning Trader\n",
        "\n",
        "class AI_Trader():  \n",
        "\n",
        "# -----------------------------------------------------------------------\n",
        "\n",
        "  # CONSTRUTOR\n",
        "\n",
        "  def __init__(self, state_size, action_space=3, model_name=\"AITrader\"):\n",
        "    \n",
        "    self.state_size = state_size # Tamanho da entrada da rede neural \n",
        "    self.action_space = action_space # Espaço de ação será 3, Comprar, Vender, Sem Ação (Tamanho da saída da rede neural)\n",
        "    self.memory = deque(maxlen=2000) # Memória com 2000 posições. A função Deque permite adicionar elementos ao final, enquanto remove elementos do início.\n",
        "    self.inventory = [] # Terá as comprar que já fizemos\n",
        "    self.model_name = model_name # Nome do modelo para o Keras\n",
        "    \n",
        "    self.gamma = 0.95 # Parâmetro que ajudará a maximizar a recompensa\n",
        "    self.epsilon = 1.0 # Taxa de aleatoriedade para atitudes ganacioas do algorítimo.\n",
        "    self.epsilon_final = 0.01 # Taxa final reduzida\n",
        "    self.epsilon_decay = 0.995 # Velocidade de decaimento da taxa\n",
        "\n",
        "    self.model = self.model_builder() # Inicializa um modelo e de rede neural e salva na classe\n",
        "\n",
        "# -----------------------------------------------------------------------\n",
        "\n",
        "  # DEFININDO A REDE NEURAL\n",
        "\n",
        "  def model_builder(self):\n",
        "        \n",
        "    model = tf.keras.models.Sequential()      \n",
        "    model.add(layers.Dense(units=32, activation='relu', input_dim=self.state_size))\n",
        "    model.add(layers.Dense(units=64, activation='relu'))\n",
        "    model.add(layers.Dense(units=128, activation='relu'))\n",
        "    model.add(layers.Dense(units=self.action_space, activation='linear')) # De maneira geral, teremos 3 saída na rede geral (número de espaços de ação)\n",
        "\n",
        "\n",
        "    model.compile(loss='mse', optimizer=keras.optimizers.Adam(learning_rate=0.001)); # Compilamos o modelo\n",
        "\n",
        "    return model # Retornamos o modelo pela função.\n",
        "\n",
        "# -----------------------------------------------------------------------\n",
        "\n",
        "  # FUNÇÃO DE TRADE\n",
        "  # Usa o Epsilon e um número aleatório para definir se usará um dado aleatório ou a previsão da rede.\n",
        "\n",
        "  def trade(self, state):\n",
        "    if random.random() <= self.epsilon:\n",
        "      return random.randrange(self.action_space) # Retonar uma resposta aleatória\n",
        "\n",
        "    actions = self.model.predict(state)\n",
        "    return np.argmax(actions[0]) # Retorna o index da maior resposta da rede\n",
        "\n",
        "# -----------------------------------------------------------------------\n",
        "\n",
        "  # LOTE DE TREINAMENTO\n",
        "\n",
        "  # Definindo o modelo para treinamento do lote\n",
        "\n",
        "  def batch_train(self, batch_size): # Função que tem o tamanho do lote como argumento\n",
        "\n",
        "    batch = [] # Iremos usar a memória como lote, por isso iniciamos com uma lista vazia\n",
        "\n",
        "    # Iteramos sobre a memória, adicionando seus elementos ao lote batch\n",
        "    for i in range(len(self.memory) - batch_size + 1, len(self.memory)): \n",
        "      batch.append(self.memory[i])\n",
        "\n",
        "    # Agora temos um lote de dados e devemos iterar sobre cada estado, recompensa,\n",
        "    # proximo_estado e conclusão do lote e treinar o modelo com isso.\n",
        "    for state, action, reward, next_state, done in batch:\n",
        "      reward = reward\n",
        "\n",
        "      # Se não estivermos no último agente da memória, então calculamos a\n",
        "      # recompensa descontando a recompensa total da recompensa atual.\n",
        "      if not done:\n",
        "        reward = reward + self.gamma * np.amax(self.model.predict(next_state)[0])\n",
        "\n",
        "      # Fazemos uma previsão e alocamos à varivel target\n",
        "      target = self.model.predict(state)\n",
        "      target[0][action] = reward\n",
        "\n",
        "      # Treinamos o modelo com o estado, usando a previsão como resultado esperado.\n",
        "      self.model.fit(state, target, epochs=1, verbose=0)\n",
        "\n",
        "    # Por fim decrementamos o epsilon a fim de gradativamente diminuir tentativas ganaciosas. \n",
        "    if self.epsilon > self.epsilon_final:\n",
        "      self.epsilon *= self.epsilon_decay\n",
        "\n",
        "# -----------------------------------------------------------------------\n",
        "\n",
        "\n",
        "# -----------------------------------------------------------------------\n",
        "\n",
        "\n",
        "# -----------------------------------------------------------------------\n",
        "\n",
        "\n",
        "# -----------------------------------------------------------------------\n",
        "    \n"
      ],
      "metadata": {
        "id": "nMWegySnF4jR"
      },
      "execution_count": None,

      "outputs": []
    },
    {
      "cell_type": "code",
      "source": [
        "# Stock Market Data Preprocessing\n",
        "\n",
        "# Definiremos algumas funções auxiliares\n",
        "\n",
        "# Sigmoid\n",
        "def sigmoid(x):\n",
        "  return 1 / (1 + math.exp(-x))\n",
        "\n",
        "# Função para formatar texto\n",
        "def stock_price_format(n):\n",
        "  if n < 0:\n",
        "    return \"- # {0:2f}\".format(abs(n))\n",
        "  else:\n",
        "    return \"$ {0:2f}\".format(abs(n))\n",
        "\n",
        "# Busca dados no Yahoo Finance\n",
        "# Formato data = \"yyyy-mm-dd\"\n",
        "def dataset_loader(stock_name, initial_date, final_date):\n",
        "\n",
        "  yfin.pdr_override()\n",
        "\n",
        "  dataset = pdr.get_data_yahoo(stock_name, start=initial_date, end=final_date)\n",
        "  \n",
        "  start_date = str(dataset.index[0]).split()[0]\n",
        "  end_date = str(dataset.index[1]).split()[0]\n",
        "  \n",
        "  close = dataset['Close']\n",
        "  \n",
        "  return close"
      ],
      "metadata": {
        "id": "mlwcYt-3fItG"
      },
      "execution_count": None,

      "outputs": []
    },
    {
      "cell_type": "markdown",
      "source": [
        "# State Creator\n",
        "\n",
        "Primeiro vamos traduzir o problema para um ambiente de aprendizado por reforço.\n",
        "\n",
        "* Cada ponto no gráfico é um ponto flutuante que representa o valor no momento do tempo.\n",
        "\n",
        "* Devemos prever o que acontecerá no próximo período de tempo, usando umas das 3 possibilidades de ação: compra, venda ou sem ação (esperar)\n",
        "\n",
        "Inicialmente vamos usar uma janela de 5 estados anteriores, para tentar prever o próximo.\n",
        "\n",
        "```windows_size = 5```\n",
        "\n",
        "Ao invés vez de prever valores reais para nosso alvo, queremos prever uma de nossas 3 ações.\n",
        "\n",
        "Em seguida, mudamos nossos estados de entrada para diferenças nos preços das ações, que representarão as mudanças de preços ao longo do tempo.\n"
      ],
      "metadata": {
        "id": "GOeEOux6ncU1"
      }
    },
    {
      "cell_type": "code",
      "source": [
        "# State Creator\n",
        "\n",
        "\n",
        "def state_creator(data, timestep, window_size):\n",
        "\n",
        "  # O index incial (starting_id) será o timestep (passos/dias que já foram dados)\n",
        "  # menos o tamanho da janela, que serão os dias olhados para trás.\n",
        "  starting_id = timestep - window_size + 1\n",
        "\n",
        "\n",
        "  # Lógica para preencher os dados vindos da tabela Data, no array windowed_data\n",
        "\n",
        "  if starting_id >= 0: # No geral este será a condição sempre executada\n",
        "    windowed_data = data[starting_id: timestep + 1]\n",
        "\n",
        "  else: # Condição executada apenas nos primeiros valores\n",
        "    windowed_data =- starting_id * [data[0]] + list(data[0:timestep + 1])\n",
        "\n",
        "  state = [] # Criação de uma array para retorno, com o estado.\n",
        "\n",
        "  for i in range(window_size - 1):\n",
        "    state.append(sigmoid(windowed_data[i + 1] - windowed_data[i]))\n",
        "\n",
        "  return np.array([state])"
      ],
      "metadata": {
        "id": "v6ZXzg8GjLM1"
      },
      "execution_count": None,

      "outputs": []
    },
    {
      "cell_type": "code",
      "source": [
        ""
      ],
      "metadata": {
        "id": "LuglY8HcbGWO"
      },
      "execution_count": None,
      "outputs": []
    },
    {
      "cell_type": "code",
      "source": [
        "# Loading a Dataset\n",
        "\n",
        "# CONFIGURAÇÕES DE IMPORTAÇÃO DE DADOS\n",
        "\n",
        "# NOME DA AÇÃO\n",
        "STOCK_NAME = \"WEGE3.SA\"\n",
        "\n",
        "# DATA INCIAL\n",
        "INITIAL_DATE = \"2021-01-01\"\n",
        "\n",
        "# DATA FINAL\n",
        "today = datetime.date.today()\n",
        "FINAL_DATE = today.strftime(\"%Y-%m-%d\") # Escolhe a data final como hoje\n",
        "\n",
        "data = dataset_loader(STOCK_NAME, INITIAL_DATE, FINAL_DATE);\n",
        "\n",
        "data"
      ],
      "metadata": {
        "colab": {
          "base_uri": "https://localhost:8080/"
        },
        "id": "subYTitYk75X",
        "outputId": "523767f8-efb9-47c6-cc5d-5c68705ce9fe"
      },
      "execution_count": None,

      "outputs": [
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            "\r[*********************100%***********************]  1 of 1 completed\n"
          ]
        },
        {
          "output_type": "execute_result",
          "data": {
            "text/plain": [
              "Date\n",
              "2021-01-04    37.310001\n",
              "2021-01-05    39.599998\n",
              "2021-01-06    40.650002\n",
              "2021-01-07    42.330002\n",
              "2021-01-08    44.889999\n",
              "                ...    \n",
              "2022-02-14    30.440001\n",
              "2022-02-15    32.880001\n",
              "2022-02-16    31.299999\n",
              "2022-02-17    30.650000\n",
              "2022-02-18    29.980000\n",
              "Name: Close, Length: 282, dtype: float64"
            ]
          },
          "metadata": {},
          "execution_count": 36
        }
      ]
    },
    {
      "cell_type": "code",
      "source": [
        "# Training the Q-Learning Trading Agent\n",
        "\n",
        "window_size = 10\n",
        "episodes = 2\n",
        "\n",
        "batch_size = 32\n",
        "data_samples = len(data) - 1\n",
        "\n",
        "trader = AI_Trader(window_size)\n",
        "trader.model.summary()"
      ],
      "metadata": {
        "id": "lNC1nFOYsk20",
        "colab": {
          "base_uri": "https://localhost:8080/"
        },
        "outputId": "b059591c-5b06-4119-bb77-c58910c1bce5"
      },
     "execution_count": None,

      "outputs": [
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            "Model: \"sequential_2\"\n",
            "_________________________________________________________________\n",
            " Layer (type)                Output Shape              Param #   \n",
            "=================================================================\n",
            " dense_8 (Dense)             (None, 32)                352       \n",
            "                                                                 \n",
            " dense_9 (Dense)             (None, 64)                2112      \n",
            "                                                                 \n",
            " dense_10 (Dense)            (None, 128)               8320      \n",
            "                                                                 \n",
            " dense_11 (Dense)            (None, 3)                 387       \n",
            "                                                                 \n",
            "=================================================================\n",
            "Total params: 11,171\n",
            "Trainable params: 11,171\n",
            "Non-trainable params: 0\n",
            "_________________________________________________________________\n"
          ]
        }
      ]
    },
    {
      "cell_type": "code",
      "source": [
        "# Defining a Training Loop\n",
        "\n",
        "# Vamos iterar sobre todos episódios\n",
        "\n",
        "for episode in range(1, episodes + 1):\n",
        "  \n",
        "  print(\"Episode: {}/{}\".format(episode, episodes))\n",
        "  \n",
        "  # Criamos o primeiro estado.\n",
        "  state = state_creator(data, 0, window_size + 1)\n",
        "  \n",
        "  # Inicializamos algumas variáveis\n",
        "  total_profit = 0\n",
        "  trader.inventory = []\n",
        "\n",
        "  # O Loop de treinamento que será executado durante uma época inteira\n",
        "  for t in tqdm(range(data_samples)):\n",
        "    \n",
        "    # O IA executa a função trade, que responderá com a ação que deve ser tomada\n",
        "    action = trader.trade(state) \n",
        "    \n",
        "    # Já definimos o próximo estado\n",
        "    # Note que o definimos com t+1, pois estamos considerando o próximo\n",
        "    # valor da ação no index da tabela de dados.\n",
        "    next_state = state_creator(data, t+1, window_size + 1)\n",
        "    \n",
        "    # Sem recompensas até agora\n",
        "    reward = 0\n",
        "\n",
        "    # Sem ação\n",
        "    if action == 0: \n",
        "      # Apenas um print e Recompensa = 0\n",
        "      print(\" - Sem ação | Total de papeis no portfolio = \", len(trader.inventory))\n",
        "    \n",
        "    # Compra\n",
        "    if action == 1:\n",
        "      # Recompensa = 0\n",
        "\n",
        "      # Adicionamos a ação comprada na array de portfolio\n",
        "      trader.inventory.append(data[t])\n",
        "\n",
        "      print(\" - AI Trader Comprou: \", stock_price_format(data[t]))\n",
        "      \n",
        "    # Venda (Deve possuir ações no portfolio)\n",
        "    elif action == 2 and len(trader.inventory) > 0: \n",
        "      \n",
        "      # Remove última ação do portfólio e a retorna\n",
        "      buy_price = trader.inventory.pop(0) \n",
        "      \n",
        "      # Recompensa = lucro ou 0 se houve prejuízo.\n",
        "      reward = max(data[t] - buy_price, 0)\n",
        "\n",
        "      total_profit += data[t] - buy_price # Soma ao lucro/prejuízo total\n",
        "\n",
        "      print(\" - AI Trader Vendeu: \", stock_price_format(data[t]), \" - Lucro: \" + stock_price_format(data[t] - buy_price) )\n",
        "\n",
        "\n",
        "    # Verifica se estamos no final de uma época\n",
        "    if t == data_samples - 1:\n",
        "      done = True\n",
        "    else:\n",
        "      done = False\n",
        "\n",
        "\n",
        "    # Salvamos os dados na memória, na mesma ordem que na função BATCH_TRAIN\n",
        "    trader.memory.append((state, action, reward, next_state, done))\n",
        "    \n",
        "    # Definimos que o estado atual é o próximo estado calculado anteriormente\n",
        "    state = next_state\n",
        "    \n",
        "    if done:\n",
        "      print(\"########################\")\n",
        "      print(\"TOTAL PROFIT: {}\".format(total_profit))\n",
        "      print(\"########################\")\n",
        "\n",
        "\n",
        "    # Se o tamanho da memória for maior que o tamanho do lote que definimos\n",
        "    # Então vamos treinar a rede, passando o tamanho do lote como argumento\n",
        "    if len(trader.memory) > batch_size:\n",
        "      trader.batch_train(batch_size)\n",
        "      \n",
        "  # A Cada 10 episódios treinados, salvamos a rede\n",
        "  if episode % 10 == 0:\n",
        "    trader.model.save(\"ai_trader_{}.h5\".format(episode))\n",
        "    "
      ],
      "metadata": {
        "colab": {
          "base_uri": "https://localhost:8080/"
        },
        "id": "_nZd_fkru7y3",
        "outputId": "1a9ca9e9-ac0b-4126-96d0-c25c883929cc"
      },
      "execution_count": None,

      "outputs": [
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            "Episode: 1/2\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r  0%|          | 0/281 [00:00<?, ?it/s]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n",
            " - Sem ação | Total de papeis no portfolio =  0\n",
            " - Sem ação | Total de papeis no portfolio =  0\n",
            " - Sem ação | Total de papeis no portfolio =  0\n",
            " - Sem ação | Total de papeis no portfolio =  0\n",
            " - Sem ação | Total de papeis no portfolio =  0\n",
            " - AI Trader Comprou:  $ 44.799999\n",
            " - Sem ação | Total de papeis no portfolio =  1\n",
            " - AI Trader Comprou:  $ 43.825001\n",
            " - AI Trader Vendeu:  $ 44.485001  - Lucro: - # 0.314999\n",
            " - Sem ação | Total de papeis no portfolio =  1\n",
            " - AI Trader Comprou:  $ 44.345001\n",
            " - Sem ação | Total de papeis no portfolio =  2\n",
            " - Sem ação | Total de papeis no portfolio =  2\n",
            " - AI Trader Vendeu:  $ 42.270000  - Lucro: - # 1.555000\n",
            " - AI Trader Vendeu:  $ 42.615002  - Lucro: - # 1.730000\n",
            " - AI Trader Comprou:  $ 44.334999\n",
            " - AI Trader Vendeu:  $ 42.785000  - Lucro: - # 1.549999\n",
            " - Sem ação | Total de papeis no portfolio =  0\n",
            " - AI Trader Comprou:  $ 42.505001\n",
            " - Sem ação | Total de papeis no portfolio =  1\n",
            " - Sem ação | Total de papeis no portfolio =  1\n",
            " - AI Trader Vendeu:  $ 43.334999  - Lucro: $ 0.829998\n",
            " - AI Trader Comprou:  $ 41.360001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 12%|█▏        | 33/281 [00:05<00:41,  5.95it/s]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 41.830002\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 12%|█▏        | 34/281 [00:10<01:27,  2.84it/s]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 43.349998\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 12%|█▏        | 35/281 [00:14<02:25,  1.69it/s]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 40.000000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 13%|█▎        | 36/281 [00:19<03:39,  1.12it/s]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 39.025002  - Lucro: - # 2.334999\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 13%|█▎        | 37/281 [00:24<05:07,  1.26s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 39.000000  - Lucro: - # 2.830002\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 14%|█▎        | 38/281 [00:28<06:44,  1.67s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 38.529999  - Lucro: - # 4.820000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 14%|█▍        | 39/281 [00:33<08:38,  2.14s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  1\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 14%|█▍        | 40/281 [00:38<10:25,  2.59s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  1\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 15%|█▍        | 41/281 [00:43<11:59,  3.00s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  1\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 15%|█▍        | 42/281 [00:47<13:21,  3.35s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 34.575001  - Lucro: - # 5.424999\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 15%|█▌        | 43/281 [00:52<14:32,  3.67s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 35.750000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 16%|█▌        | 44/281 [00:57<15:33,  3.94s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  1\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 16%|█▌        | 45/281 [01:01<16:17,  4.14s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 35.959999  - Lucro: $ 0.209999\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 16%|█▋        | 46/281 [01:06<16:47,  4.29s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 35.575001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 17%|█▋        | 47/281 [01:11<17:08,  4.40s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 36.610001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 17%|█▋        | 48/281 [01:16<17:42,  4.56s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  2\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 17%|█▋        | 49/281 [01:20<17:44,  4.59s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 36.595001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 18%|█▊        | 50/281 [01:25<17:45,  4.61s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 35.904999  - Lucro: $ 0.329998\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 18%|█▊        | 51/281 [01:30<17:52,  4.66s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 35.974998\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 19%|█▊        | 52/281 [01:34<17:47,  4.66s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  3\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 19%|█▉        | 53/281 [01:39<17:44,  4.67s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 35.805000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 19%|█▉        | 54/281 [01:44<17:41,  4.68s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 35.259998\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 20%|█▉        | 55/281 [01:49<17:42,  4.70s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 35.535000  - Lucro: - # 1.075001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 20%|█▉        | 56/281 [01:53<17:53,  4.77s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 36.369999\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 20%|██        | 57/281 [01:58<17:48,  4.77s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 36.049999\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 21%|██        | 58/281 [02:03<17:46,  4.78s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 37.525002  - Lucro: $ 0.930000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 21%|██        | 59/281 [02:08<17:39,  4.77s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  5\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 21%|██▏       | 60/281 [02:13<17:31,  4.76s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  5\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 22%|██▏       | 61/281 [02:17<17:22,  4.74s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  5\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 22%|██▏       | 62/281 [02:22<17:28,  4.79s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 38.549999  - Lucro: $ 2.575001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 22%|██▏       | 63/281 [02:27<17:26,  4.80s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  4\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 23%|██▎       | 64/281 [02:32<17:15,  4.77s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  4\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 23%|██▎       | 65/281 [02:37<17:21,  4.82s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  4\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 23%|██▎       | 66/281 [02:41<17:15,  4.82s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 37.755001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 24%|██▍       | 67/281 [02:46<17:14,  4.84s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 37.490002  - Lucro: $ 1.685001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 24%|██▍       | 68/281 [02:51<17:22,  4.90s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  4\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 25%|██▍       | 69/281 [02:56<17:08,  4.85s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 39.150002\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 25%|██▍       | 70/281 [03:01<17:07,  4.87s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 38.525002\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 25%|██▌       | 71/281 [03:06<16:57,  4.84s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 37.900002\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 26%|██▌       | 72/281 [03:11<16:53,  4.85s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 36.935001  - Lucro: $ 1.675003\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 26%|██▌       | 73/281 [03:16<17:07,  4.94s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 36.439999\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 26%|██▋       | 74/281 [03:21<17:05,  4.96s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  7\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 27%|██▋       | 75/281 [03:26<17:00,  4.95s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 37.314999  - Lucro: $ 0.945000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 27%|██▋       | 76/281 [03:31<16:54,  4.95s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 37.009998  - Lucro: $ 0.959999\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 27%|██▋       | 77/281 [03:36<16:54,  4.97s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 36.500000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 28%|██▊       | 78/281 [03:41<16:49,  4.97s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 35.490002  - Lucro: - # 2.264999\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 28%|██▊       | 79/281 [03:46<16:44,  4.97s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 35.009998  - Lucro: - # 4.140003\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 28%|██▊       | 80/281 [03:51<16:39,  4.97s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 34.450001  - Lucro: - # 4.075001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 29%|██▉       | 81/281 [03:55<16:21,  4.91s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 33.700001  - Lucro: - # 4.200001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 29%|██▉       | 82/281 [04:00<16:27,  4.96s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 34.709999  - Lucro: - # 1.730000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 30%|██▉       | 83/281 [04:05<16:11,  4.91s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 33.730000  - Lucro: - # 2.770000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            " 30%|███       | 85/281 [04:15<15:48,  4.84s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 31%|███       | 86/281 [04:20<15:38,  4.81s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            " 32%|███▏      | 90/281 [04:39<15:25,  4.85s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 32.169998\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 32%|███▏      | 91/281 [04:44<15:18,  4.83s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  1\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 33%|███▎      | 92/281 [04:48<15:08,  4.81s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 31.600000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 33%|███▎      | 93/281 [04:53<15:02,  4.80s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 31.860001  - Lucro: - # 0.309998\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 33%|███▎      | 94/281 [04:58<14:53,  4.78s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  1\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 34%|███▍      | 95/281 [05:03<14:45,  4.76s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 32.650002  - Lucro: $ 1.050001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 34%|███▍      | 96/281 [05:07<14:34,  4.73s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 32.970001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 35%|███▍      | 97/281 [05:12<14:28,  4.72s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 32.799999\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 35%|███▍      | 98/281 [05:17<14:28,  4.74s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 32.810001  - Lucro: - # 0.160000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 35%|███▌      | 99/281 [05:22<14:32,  4.80s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 34.400002\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 36%|███▌      | 100/281 [05:26<14:26,  4.79s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 34.150002  - Lucro: $ 1.350002\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 36%|███▌      | 101/281 [05:31<14:17,  4.77s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 34.189999  - Lucro: - # 0.210003\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            " 37%|███▋      | 104/281 [05:45<13:50,  4.69s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 37%|███▋      | 105/281 [05:50<13:46,  4.70s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            " 38%|███▊      | 107/281 [05:59<13:34,  4.68s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 38%|███▊      | 108/281 [06:04<13:46,  4.78s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 34.209999\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 39%|███▉      | 109/281 [06:09<13:44,  4.79s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 34.880001  - Lucro: $ 0.670002\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            " 40%|███▉      | 111/281 [06:19<13:39,  4.82s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 34.830002\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 40%|███▉      | 112/281 [06:24<13:33,  4.81s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 34.419998\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 40%|████      | 113/281 [06:28<13:36,  4.86s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  2\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 41%|████      | 114/281 [06:33<13:26,  4.83s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 35.250000  - Lucro: $ 0.419998\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 41%|████      | 115/281 [06:38<13:22,  4.84s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 34.290001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 41%|████▏     | 116/281 [06:43<13:35,  4.94s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 33.950001  - Lucro: - # 0.469997\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 42%|████▏     | 117/281 [06:48<13:26,  4.92s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 34.330002  - Lucro: $ 0.040001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            " 42%|████▏     | 119/281 [06:58<13:22,  4.95s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 34.000000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 43%|████▎     | 120/281 [07:03<13:18,  4.96s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 34.700001  - Lucro: $ 0.700001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 43%|████▎     | 121/281 [07:08<13:11,  4.94s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 43%|████▎     | 122/281 [07:13<13:00,  4.91s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 34.250000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 44%|████▍     | 123/281 [07:18<12:56,  4.92s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 35.259998  - Lucro: $ 1.009998\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 44%|████▍     | 124/281 [07:23<12:50,  4.91s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 34.910000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 44%|████▍     | 125/281 [07:28<12:53,  4.96s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  1\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 45%|████▍     | 126/281 [07:33<12:43,  4.92s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 36.060001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 45%|████▌     | 127/281 [07:37<12:35,  4.91s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 34.730000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 46%|████▌     | 128/281 [07:42<12:23,  4.86s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 35.580002\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 46%|████▌     | 129/281 [07:47<12:14,  4.83s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  4\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 46%|████▋     | 130/281 [07:52<12:15,  4.87s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  4\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 47%|████▋     | 131/281 [07:57<12:15,  4.91s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 34.919998  - Lucro: $ 0.009998\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 47%|████▋     | 132/281 [08:02<12:04,  4.86s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 34.290001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 47%|████▋     | 133/281 [08:07<12:12,  4.95s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 34.180000  - Lucro: - # 1.880001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 48%|████▊     | 134/281 [08:12<12:10,  4.97s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 34.549999\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 48%|████▊     | 135/281 [08:17<12:04,  4.97s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 34.450001  - Lucro: - # 0.279999\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 48%|████▊     | 136/281 [08:22<11:57,  4.95s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 35.099998  - Lucro: - # 0.480003\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 49%|████▉     | 137/281 [08:27<11:53,  4.95s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  2\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 49%|████▉     | 138/281 [08:32<11:46,  4.94s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 34.650002\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 49%|████▉     | 139/281 [08:36<11:34,  4.89s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 34.389999  - Lucro: $ 0.099998\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 50%|████▉     | 140/281 [08:41<11:30,  4.89s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 37.200001  - Lucro: $ 2.650002\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 50%|█████     | 141/281 [08:46<11:29,  4.93s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 36.130001  - Lucro: $ 1.480000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 51%|█████     | 142/281 [08:51<11:34,  5.00s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 51%|█████     | 143/281 [08:56<11:22,  4.95s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 51%|█████     | 144/281 [09:01<11:17,  4.95s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 36.419998\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 52%|█████▏    | 145/281 [09:06<11:07,  4.91s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 36.070000  - Lucro: - # 0.349998\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 52%|█████▏    | 146/281 [09:11<11:03,  4.91s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 35.860001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 52%|█████▏    | 147/281 [09:16<10:58,  4.91s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 36.240002  - Lucro: $ 0.380001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            " 54%|█████▍    | 153/281 [09:45<10:21,  4.86s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 55%|█████▍    | 154/281 [09:50<10:15,  4.85s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 34.689999\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 55%|█████▌    | 155/281 [09:55<10:10,  4.85s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 33.580002  - Lucro: - # 1.109997\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 56%|█████▌    | 156/281 [09:59<10:06,  4.85s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            " 56%|█████▌    | 158/281 [10:09<09:54,  4.83s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 35.750000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 57%|█████▋    | 159/281 [10:14<09:58,  4.91s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 35.660000  - Lucro: - # 0.090000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            " 57%|█████▋    | 161/281 [10:24<09:40,  4.84s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 58%|█████▊    | 162/281 [10:29<09:41,  4.89s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            " 61%|██████    | 171/281 [11:13<09:06,  4.97s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            " 62%|██████▏   | 173/281 [11:23<08:56,  4.96s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 39.380001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 62%|██████▏   | 174/281 [11:28<08:47,  4.93s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 39.840000  - Lucro: $ 0.459999\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            " 64%|██████▍   | 180/281 [11:58<08:14,  4.90s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            " 65%|██████▍   | 182/281 [12:07<08:05,  4.91s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            " 65%|██████▌   | 184/281 [12:17<07:54,  4.89s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 39.869999\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 66%|██████▌   | 185/281 [12:22<07:55,  4.96s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 39.630001  - Lucro: - # 0.239998\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            " 67%|██████▋   | 188/281 [12:37<07:38,  4.93s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 37.529999\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 67%|██████▋   | 189/281 [12:42<07:32,  4.92s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 37.509998  - Lucro: - # 0.020000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            " 68%|██████▊   | 191/281 [12:52<07:19,  4.88s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 38.650002\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 68%|██████▊   | 192/281 [12:57<07:13,  4.87s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 38.619999  - Lucro: - # 0.030003\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            " 69%|██████▉   | 195/281 [13:12<07:03,  4.93s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            " 70%|███████   | 198/281 [13:27<06:56,  5.02s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 40.119999\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 71%|███████   | 199/281 [13:32<06:50,  5.01s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 39.349998\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 71%|███████   | 200/281 [13:37<06:47,  5.03s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 38.889999\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 72%|███████▏  | 201/281 [13:42<06:42,  5.03s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 39.549999\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 72%|███████▏  | 202/281 [13:47<06:43,  5.10s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 39.660000  - Lucro: - # 0.459999\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 72%|███████▏  | 203/281 [13:52<06:38,  5.11s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 39.200001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 73%|███████▎  | 204/281 [13:57<06:29,  5.06s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 37.830002\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 73%|███████▎  | 205/281 [14:02<06:24,  5.06s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 37.000000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 73%|███████▎  | 206/281 [14:07<06:22,  5.10s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 36.959999\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 74%|███████▎  | 207/281 [14:13<06:19,  5.13s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 37.680000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 74%|███████▍  | 208/281 [14:18<06:09,  5.06s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 37.720001  - Lucro: - # 1.629997\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 74%|███████▍  | 209/281 [14:22<06:01,  5.02s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 37.680000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 75%|███████▍  | 210/281 [14:27<05:54,  4.99s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 37.099998\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 75%|███████▌  | 211/281 [14:33<05:53,  5.05s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 36.790001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 75%|███████▌  | 212/281 [14:38<05:46,  5.01s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  10\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 76%|███████▌  | 213/281 [14:42<05:39,  4.99s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 36.500000  - Lucro: - # 2.389999\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 76%|███████▌  | 214/281 [14:47<05:32,  4.97s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  9\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 77%|███████▋  | 215/281 [14:52<05:25,  4.93s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 35.380001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 77%|███████▋  | 216/281 [14:57<05:20,  4.93s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 35.770000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 77%|███████▋  | 217/281 [15:02<05:17,  4.96s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 35.919998\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 78%|███████▊  | 218/281 [15:07<05:11,  4.94s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 35.790001  - Lucro: - # 3.759998\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 78%|███████▊  | 219/281 [15:12<05:09,  5.00s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 34.669998\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 78%|███████▊  | 220/281 [15:17<05:04,  4.99s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 34.160000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 79%|███████▊  | 221/281 [15:22<04:58,  4.97s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 33.529999  - Lucro: - # 5.670002\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 79%|███████▉  | 222/281 [15:27<04:50,  4.92s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  12\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 79%|███████▉  | 223/281 [15:32<04:45,  4.92s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  12\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 80%|███████▉  | 224/281 [15:37<04:41,  4.93s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  12\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 80%|████████  | 225/281 [15:42<04:34,  4.91s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  12\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 80%|████████  | 226/281 [15:46<04:29,  4.89s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  12\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 81%|████████  | 227/281 [15:51<04:25,  4.91s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 32.720001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 81%|████████  | 228/281 [15:57<04:24,  4.99s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  13\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 81%|████████▏ | 229/281 [16:01<04:17,  4.96s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 33.009998\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 82%|████████▏ | 230/281 [16:06<04:10,  4.92s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  14\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 82%|████████▏ | 231/281 [16:11<04:09,  4.99s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 36.040001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 83%|████████▎ | 232/281 [16:17<04:06,  5.03s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  15\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 83%|████████▎ | 233/281 [16:22<04:02,  5.06s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  15\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 83%|████████▎ | 234/281 [16:27<03:56,  5.03s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  15\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 84%|████████▎ | 235/281 [16:32<03:49,  4.99s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 35.290001  - Lucro: - # 2.540001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 84%|████████▍ | 236/281 [16:37<03:46,  5.04s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 35.610001  - Lucro: - # 1.389999\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 84%|████████▍ | 237/281 [16:42<03:40,  5.01s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 35.599998\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 85%|████████▍ | 238/281 [16:47<03:34,  5.00s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  14\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 85%|████████▌ | 239/281 [16:52<03:30,  5.00s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 34.009998\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 85%|████████▌ | 240/281 [16:57<03:24,  4.99s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  15\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 86%|████████▌ | 241/281 [17:02<03:19,  4.99s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 33.669998  - Lucro: - # 3.290001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 86%|████████▌ | 242/281 [17:07<03:14,  4.98s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 33.470001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 86%|████████▋ | 243/281 [17:11<03:07,  4.93s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  15\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 87%|████████▋ | 244/281 [17:16<03:02,  4.93s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  15\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 87%|████████▋ | 245/281 [17:22<03:00,  5.02s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  15\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 88%|████████▊ | 246/281 [17:26<02:54,  4.99s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 32.980000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 88%|████████▊ | 247/281 [17:31<02:49,  5.00s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 32.020000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 88%|████████▊ | 248/281 [17:36<02:43,  4.96s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 31.860001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 89%|████████▊ | 249/281 [17:41<02:39,  4.98s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 30.180000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 89%|████████▉ | 250/281 [17:46<02:32,  4.92s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 30.170000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 89%|████████▉ | 251/281 [17:51<02:27,  4.91s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 29.410000  - Lucro: - # 8.270000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 90%|████████▉ | 252/281 [17:56<02:21,  4.88s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 28.559999\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 90%|█████████ | 253/281 [18:01<02:17,  4.92s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 29.040001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 90%|█████████ | 254/281 [18:06<02:15,  5.01s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 30.219999  - Lucro: - # 7.460001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 91%|█████████ | 255/281 [18:11<02:10,  5.00s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 29.100000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 91%|█████████ | 256/281 [18:16<02:05,  5.01s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 30.700001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 91%|█████████▏| 257/281 [18:21<01:59,  4.97s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 30.350000  - Lucro: - # 6.749998\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 92%|█████████▏| 258/281 [18:26<01:53,  4.94s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  21\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 92%|█████████▏| 259/281 [18:31<01:48,  4.94s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  21\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 93%|█████████▎| 260/281 [18:36<01:43,  4.93s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  21\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 93%|█████████▎| 261/281 [18:41<01:38,  4.95s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 30.610001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 93%|█████████▎| 262/281 [18:46<01:35,  5.02s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  22\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 94%|█████████▎| 263/281 [18:51<01:30,  5.00s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 30.200001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 94%|█████████▍| 264/281 [18:56<01:24,  4.95s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 30.740000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 94%|█████████▍| 265/281 [19:01<01:18,  4.92s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 32.400002\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 95%|█████████▍| 266/281 [19:05<01:13,  4.91s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  25\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 95%|█████████▌| 267/281 [19:11<01:09,  5.00s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  25\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 95%|█████████▌| 268/281 [19:16<01:05,  5.07s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 32.060001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 96%|█████████▌| 269/281 [19:21<01:01,  5.09s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  26\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 96%|█████████▌| 270/281 [19:26<00:55,  5.07s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 30.950001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 96%|█████████▋| 271/281 [19:31<00:51,  5.13s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  27\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 97%|█████████▋| 272/281 [19:36<00:45,  5.07s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 31.000000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 97%|█████████▋| 273/281 [19:41<00:40,  5.03s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  28\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 98%|█████████▊| 274/281 [19:46<00:35,  5.03s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 30.290001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 98%|█████████▊| 275/281 [19:51<00:29,  4.98s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  29\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 98%|█████████▊| 276/281 [19:56<00:24,  4.95s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  29\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 99%|█████████▊| 277/281 [20:01<00:19,  4.97s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  29\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 99%|█████████▉| 278/281 [20:06<00:14,  4.93s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  29\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 99%|█████████▉| 279/281 [20:11<00:09,  4.98s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 31.299999\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r100%|█████████▉| 280/281 [20:16<00:04,  4.98s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 30.650000\n",
            "########################\n",
            "TOTAL PROFIT: -69.59499549865723\n",
            "########################\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "100%|██████████| 281/281 [20:21<00:00,  4.35s/it]\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            "Episode: 2/2\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r  0%|          | 0/281 [00:00<?, ?it/s]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r  0%|          | 1/281 [00:04<22:52,  4.90s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r  1%|          | 2/281 [00:09<22:35,  4.86s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "  2%|▏         | 5/281 [00:24<22:31,  4.90s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "  2%|▏         | 7/281 [00:34<22:35,  4.95s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "  3%|▎         | 9/281 [00:44<22:12,  4.90s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r  4%|▎         | 10/281 [00:49<22:14,  4.92s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "  4%|▍         | 12/281 [00:58<21:52,  4.88s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r  5%|▍         | 13/281 [01:03<22:04,  4.94s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "  5%|▌         | 15/281 [01:13<22:01,  4.97s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 44.474998\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r  6%|▌         | 16/281 [01:18<21:52,  4.95s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  1\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r  6%|▌         | 17/281 [01:23<21:50,  4.96s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 43.985001  - Lucro: - # 0.489998\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r  6%|▋         | 18/281 [01:28<21:35,  4.93s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 41.895000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r  7%|▋         | 19/281 [01:33<21:22,  4.89s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 42.270000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r  7%|▋         | 20/281 [01:38<21:11,  4.87s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 42.615002\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r  7%|▋         | 21/281 [01:43<21:11,  4.89s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 44.334999  - Lucro: $ 2.439999\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r  8%|▊         | 22/281 [01:48<21:16,  4.93s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  2\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r  8%|▊         | 23/281 [01:53<21:35,  5.02s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  2\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r  9%|▊         | 24/281 [01:58<21:48,  5.09s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 42.750000  - Lucro: $ 0.480000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r  9%|▉         | 25/281 [02:03<21:29,  5.04s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 42.505001  - Lucro: - # 0.110001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r  9%|▉         | 26/281 [02:08<21:25,  5.04s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 10%|▉         | 27/281 [02:13<21:15,  5.02s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            " 10%|█         | 29/281 [02:23<20:56,  4.99s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 41.974998\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 11%|█         | 30/281 [02:28<20:47,  4.97s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 41.689999  - Lucro: - # 0.285000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            " 12%|█▏        | 35/281 [02:53<20:33,  5.02s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 40.000000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 13%|█▎        | 36/281 [02:58<20:21,  4.98s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 39.025002  - Lucro: - # 0.974998\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 13%|█▎        | 37/281 [03:03<20:05,  4.94s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 39.000000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 14%|█▎        | 38/281 [03:08<20:00,  4.94s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 38.529999  - Lucro: - # 0.470001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            " 15%|█▍        | 42/281 [03:28<19:44,  4.95s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 34.575001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 15%|█▌        | 43/281 [03:33<19:39,  4.95s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 35.750000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 16%|█▌        | 44/281 [03:38<19:35,  4.96s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 36.139999\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 16%|█▌        | 45/281 [03:43<19:34,  4.98s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 35.959999  - Lucro: $ 1.384998\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 16%|█▋        | 46/281 [03:48<19:24,  4.95s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 35.575001  - Lucro: - # 0.174999\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 17%|█▋        | 47/281 [03:52<19:16,  4.94s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  1\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 17%|█▋        | 48/281 [03:57<19:20,  4.98s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 36.700001  - Lucro: $ 0.560001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 17%|█▋        | 49/281 [04:02<19:06,  4.94s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 36.595001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 18%|█▊        | 50/281 [04:08<19:19,  5.02s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  1\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 18%|█▊        | 51/281 [04:13<19:14,  5.02s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  1\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 19%|█▊        | 52/281 [04:18<19:05,  5.00s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 35.650002\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 19%|█▉        | 53/281 [04:22<18:52,  4.97s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 35.805000  - Lucro: - # 0.790001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 19%|█▉        | 54/281 [04:27<18:44,  4.95s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  1\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 20%|█▉        | 55/281 [04:32<18:36,  4.94s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 35.535000  - Lucro: - # 0.115002\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 20%|█▉        | 56/281 [04:37<18:24,  4.91s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 36.369999\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 20%|██        | 57/281 [04:42<18:33,  4.97s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 36.049999  - Lucro: - # 0.320000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            " 21%|██        | 59/281 [04:53<18:48,  5.08s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 37.279999\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 21%|██▏       | 60/281 [04:58<18:32,  5.03s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 37.529999  - Lucro: $ 0.250000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 22%|██▏       | 61/281 [05:03<18:23,  5.02s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 22%|██▏       | 62/281 [05:08<18:16,  5.01s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 38.549999\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 22%|██▏       | 63/281 [05:12<18:06,  4.98s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  1\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 23%|██▎       | 64/281 [05:17<17:53,  4.95s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  1\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 23%|██▎       | 65/281 [05:22<17:46,  4.94s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  1\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 23%|██▎       | 66/281 [05:27<17:44,  4.95s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 37.755001  - Lucro: - # 0.794998\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 24%|██▍       | 67/281 [05:32<17:53,  5.02s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            " 25%|██▍       | 69/281 [05:42<17:43,  5.02s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 39.150002\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 25%|██▍       | 70/281 [05:47<17:26,  4.96s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 38.525002  - Lucro: - # 0.625000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 25%|██▌       | 71/281 [05:52<17:16,  4.94s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 26%|██▌       | 72/281 [05:57<17:11,  4.93s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 26%|██▌       | 73/281 [06:02<17:04,  4.93s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 26%|██▋       | 74/281 [06:07<17:05,  4.95s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 37.605000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 27%|██▋       | 75/281 [06:12<17:15,  5.03s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 37.314999\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 27%|██▋       | 76/281 [06:17<17:15,  5.05s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 37.009998  - Lucro: - # 0.595001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 27%|██▋       | 77/281 [06:22<17:09,  5.05s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  1\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 28%|██▊       | 78/281 [06:27<17:04,  5.05s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  1\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 28%|██▊       | 79/281 [06:32<17:04,  5.07s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  1\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 28%|██▊       | 80/281 [06:38<17:02,  5.09s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  1\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 29%|██▉       | 81/281 [06:43<16:59,  5.10s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  1\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 29%|██▉       | 82/281 [06:48<16:57,  5.11s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 34.709999  - Lucro: - # 2.605000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            " 31%|███       | 86/281 [07:08<16:38,  5.12s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 31%|███       | 87/281 [07:13<16:25,  5.08s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            " 35%|███▍      | 97/281 [08:03<15:23,  5.02s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 35%|███▍      | 98/281 [08:08<15:24,  5.05s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 35%|███▌      | 99/281 [08:13<15:16,  5.04s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 36%|███▌      | 100/281 [08:18<15:12,  5.04s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 36%|███▌      | 101/281 [08:24<15:24,  5.14s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 36%|███▋      | 102/281 [08:29<15:17,  5.12s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 37%|███▋      | 103/281 [08:34<15:08,  5.10s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 37%|███▋      | 104/281 [08:39<14:57,  5.07s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 37%|███▋      | 105/281 [08:44<14:46,  5.04s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 38%|███▊      | 106/281 [08:49<14:39,  5.03s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 38%|███▊      | 107/281 [08:54<14:29,  5.00s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 38%|███▊      | 108/281 [08:59<14:21,  4.98s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 39%|███▉      | 109/281 [09:03<14:12,  4.96s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 34.880001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 39%|███▉      | 110/281 [09:09<14:22,  5.05s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 35.459999\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 40%|███▉      | 111/281 [09:14<14:15,  5.03s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  2\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 40%|███▉      | 112/281 [09:19<14:10,  5.03s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  2\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 40%|████      | 113/281 [09:24<14:05,  5.04s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  2\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 41%|████      | 114/281 [09:29<13:59,  5.03s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 35.250000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 41%|████      | 115/281 [09:34<13:46,  4.98s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  3\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 41%|████▏     | 116/281 [09:39<13:38,  4.96s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  3\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 42%|████▏     | 117/281 [09:44<13:33,  4.96s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 34.330002\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 42%|████▏     | 118/281 [09:49<13:39,  5.03s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  4\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 42%|████▏     | 119/281 [09:54<13:26,  4.98s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  4\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 43%|████▎     | 120/281 [09:59<13:20,  4.97s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 34.700001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 43%|████▎     | 121/281 [10:04<13:14,  4.96s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 33.689999\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 43%|████▎     | 122/281 [10:08<13:09,  4.96s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  6\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 44%|████▍     | 123/281 [10:14<13:07,  4.98s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 35.259998  - Lucro: $ 0.379997\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 44%|████▍     | 124/281 [10:18<13:02,  4.98s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 34.910000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 44%|████▍     | 125/281 [10:23<12:52,  4.95s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 34.549999  - Lucro: - # 0.910000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 45%|████▍     | 126/281 [10:28<12:47,  4.95s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 36.060001  - Lucro: $ 0.810001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 45%|████▌     | 127/281 [10:34<12:54,  5.03s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 34.730000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 46%|████▌     | 128/281 [10:38<12:41,  4.98s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  5\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 46%|████▌     | 129/281 [10:43<12:32,  4.95s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 35.180000  - Lucro: $ 0.849998\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 46%|████▋     | 130/281 [10:48<12:25,  4.94s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  4\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 47%|████▋     | 131/281 [10:53<12:23,  4.96s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 34.919998  - Lucro: $ 0.219997\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 47%|████▋     | 132/281 [10:58<12:33,  5.06s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  3\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 47%|████▋     | 133/281 [11:04<12:32,  5.09s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  3\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 48%|████▊     | 134/281 [11:09<12:24,  5.06s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  3\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 48%|████▊     | 135/281 [11:14<12:28,  5.13s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 34.450001  - Lucro: $ 0.760002\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 48%|████▊     | 136/281 [11:19<12:15,  5.08s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 35.099998  - Lucro: $ 0.189999\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 49%|████▉     | 137/281 [11:24<12:05,  5.04s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  1\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 49%|████▉     | 138/281 [11:29<11:57,  5.02s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 34.650002  - Lucro: - # 0.079998\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            " 51%|█████     | 144/281 [11:59<11:22,  4.98s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 52%|█████▏    | 145/281 [12:04<11:18,  4.99s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            " 53%|█████▎    | 150/281 [12:28<10:41,  4.89s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 54%|█████▎    | 151/281 [12:33<10:39,  4.92s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 54%|█████▍    | 152/281 [12:38<10:45,  5.00s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 54%|█████▍    | 153/281 [12:43<10:38,  4.99s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 55%|█████▍    | 154/281 [12:48<10:28,  4.95s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 55%|█████▌    | 155/281 [12:53<10:24,  4.96s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 33.580002\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 56%|█████▌    | 156/281 [12:58<10:18,  4.95s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 35.189999  - Lucro: $ 1.609997\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 56%|█████▌    | 157/281 [13:03<10:10,  4.92s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 35.980000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 56%|█████▌    | 158/281 [13:07<10:01,  4.89s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 35.750000  - Lucro: - # 0.230000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            " 57%|█████▋    | 160/281 [13:17<09:53,  4.90s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 36.380001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 57%|█████▋    | 161/281 [13:22<09:57,  4.98s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 36.770000  - Lucro: $ 0.389999\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            " 59%|█████▊    | 165/281 [13:42<09:29,  4.91s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 59%|█████▉    | 166/281 [13:47<09:26,  4.93s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 35.740002\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 59%|█████▉    | 167/281 [13:52<09:27,  4.98s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 36.250000  - Lucro: $ 0.509998\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 60%|█████▉    | 168/281 [13:57<09:27,  5.02s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 36.380001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 60%|██████    | 169/281 [14:03<09:31,  5.10s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 36.490002  - Lucro: $ 0.110001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            " 61%|██████    | 171/281 [14:13<09:16,  5.06s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 38.660000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 61%|██████    | 172/281 [14:18<09:08,  5.03s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 38.900002  - Lucro: $ 0.240002\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 62%|██████▏   | 173/281 [14:23<09:04,  5.04s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 39.380001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 62%|██████▏   | 174/281 [14:28<08:56,  5.01s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 39.840000  - Lucro: $ 0.459999\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 62%|██████▏   | 175/281 [14:33<08:49,  5.00s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 39.900002\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 63%|██████▎   | 176/281 [14:38<08:43,  4.98s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 39.189999\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 63%|██████▎   | 177/281 [14:42<08:36,  4.97s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  2\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 63%|██████▎   | 178/281 [14:48<08:38,  5.03s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 39.750000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 64%|██████▎   | 179/281 [14:53<08:29,  4.99s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 40.570000  - Lucro: $ 0.669998\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 64%|██████▍   | 180/281 [14:58<08:23,  4.99s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 41.169998  - Lucro: $ 1.980000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 64%|██████▍   | 181/281 [15:02<08:15,  4.96s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 41.340000  - Lucro: $ 1.590000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            " 65%|██████▌   | 184/281 [15:17<08:00,  4.95s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            " 69%|██████▉   | 194/281 [16:07<07:07,  4.91s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            " 70%|██████▉   | 196/281 [16:17<07:00,  4.95s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 70%|███████   | 197/281 [16:21<06:54,  4.93s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 39.349998\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 70%|███████   | 198/281 [16:26<06:51,  4.96s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 40.119999  - Lucro: $ 0.770000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            " 73%|███████▎  | 204/281 [16:57<06:32,  5.09s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 73%|███████▎  | 205/281 [17:01<06:22,  5.04s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 73%|███████▎  | 206/281 [17:06<06:16,  5.02s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 36.959999\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 74%|███████▎  | 207/281 [17:11<06:09,  4.99s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 37.680000  - Lucro: $ 0.720001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            " 75%|███████▍  | 210/281 [17:26<05:49,  4.92s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 75%|███████▌  | 211/281 [17:31<05:43,  4.91s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 75%|███████▌  | 212/281 [17:36<05:45,  5.00s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 76%|███████▌  | 213/281 [17:41<05:39,  4.99s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 76%|███████▌  | 214/281 [17:46<05:32,  4.96s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 35.650002\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 77%|███████▋  | 215/281 [17:51<05:26,  4.95s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 35.380001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 77%|███████▋  | 216/281 [17:56<05:18,  4.90s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  2\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 77%|███████▋  | 217/281 [18:01<05:13,  4.90s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  2\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 78%|███████▊  | 218/281 [18:06<05:09,  4.91s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  2\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 78%|███████▊  | 219/281 [18:11<05:05,  4.93s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 34.669998\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 78%|███████▊  | 220/281 [18:16<05:01,  4.95s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  3\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 79%|███████▊  | 221/281 [18:21<04:59,  5.00s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 33.529999\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 79%|███████▉  | 222/281 [18:26<04:54,  4.99s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 33.799999\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 79%|███████▉  | 223/281 [18:30<04:47,  4.95s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 32.689999  - Lucro: - # 2.960003\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 80%|███████▉  | 224/281 [18:35<04:40,  4.92s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 32.410000  - Lucro: - # 2.970001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 80%|████████  | 225/281 [18:40<04:35,  4.92s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 32.230000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 80%|████████  | 226/281 [18:45<04:29,  4.90s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 32.490002  - Lucro: - # 2.179996\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 81%|████████  | 227/281 [18:50<04:24,  4.89s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 32.720001  - Lucro: - # 0.809998\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 81%|████████  | 228/281 [18:55<04:18,  4.89s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 32.779999\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 81%|████████▏ | 229/281 [19:00<04:18,  4.96s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  3\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 82%|████████▏ | 230/281 [19:05<04:12,  4.95s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 34.560001  - Lucro: $ 0.760002\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 82%|████████▏ | 231/281 [19:10<04:07,  4.94s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 36.040001  - Lucro: $ 3.810001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 83%|████████▎ | 232/281 [19:15<04:02,  4.95s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 36.500000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 83%|████████▎ | 233/281 [19:20<03:56,  4.93s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 35.799999  - Lucro: $ 3.020000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 83%|████████▎ | 234/281 [19:25<03:50,  4.91s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 35.599998  - Lucro: - # 0.900002\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 84%|████████▎ | 235/281 [19:29<03:45,  4.91s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 84%|████████▍ | 236/281 [19:34<03:42,  4.94s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            " 87%|████████▋ | 245/281 [20:20<02:57,  4.93s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 32.740002\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 88%|████████▊ | 246/281 [20:25<02:55,  5.01s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 32.980000  - Lucro: $ 0.239998\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 88%|████████▊ | 247/281 [20:30<02:49,  4.99s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 32.020000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 88%|████████▊ | 248/281 [20:35<02:43,  4.95s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 31.860001  - Lucro: - # 0.160000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 89%|████████▊ | 249/281 [20:40<02:39,  4.97s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 30.180000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 89%|████████▉ | 250/281 [20:45<02:34,  4.98s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 30.170000  - Lucro: - # 0.010000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 89%|████████▉ | 251/281 [20:50<02:28,  4.95s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 29.410000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 90%|████████▉ | 252/281 [20:55<02:22,  4.93s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 28.559999\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 90%|█████████ | 253/281 [20:59<02:18,  4.95s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 29.040001  - Lucro: - # 0.369999\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 90%|█████████ | 254/281 [21:04<02:12,  4.93s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 30.219999  - Lucro: $ 1.660000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            " 92%|█████████▏| 259/281 [21:30<01:50,  5.03s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  0\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            " 93%|█████████▎| 261/281 [21:40<01:40,  5.02s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 30.610001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 93%|█████████▎| 262/281 [21:45<01:35,  5.00s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 29.900000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 94%|█████████▎| 263/281 [21:50<01:31,  5.06s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - Sem ação | Total de papeis no portfolio =  2\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 94%|█████████▍| 264/281 [21:55<01:25,  5.02s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 30.740000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 94%|█████████▍| 265/281 [22:00<01:19,  5.00s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 32.400002  - Lucro: $ 1.790001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 95%|█████████▍| 266/281 [22:05<01:14,  4.99s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 31.620001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 95%|█████████▌| 267/281 [22:10<01:09,  4.99s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 32.160000  - Lucro: $ 2.260000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 95%|█████████▌| 268/281 [22:15<01:04,  4.97s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 32.060001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 96%|█████████▌| 269/281 [22:20<00:59,  4.96s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 31.770000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 96%|█████████▌| 270/281 [22:25<00:54,  4.96s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 30.950001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 96%|█████████▋| 271/281 [22:30<00:49,  4.99s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 30.850000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 97%|█████████▋| 272/281 [22:35<00:45,  5.05s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 31.000000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 97%|█████████▋| 273/281 [22:40<00:40,  5.04s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 30.219999\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 98%|█████████▊| 274/281 [22:45<00:35,  5.02s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 30.290001  - Lucro: - # 0.449999\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 98%|█████████▊| 275/281 [22:50<00:30,  5.04s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 29.740000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 98%|█████████▊| 276/281 [22:55<00:25,  5.08s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 29.530001  - Lucro: - # 2.090000\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 99%|█████████▊| 277/281 [23:00<00:20,  5.15s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 30.440001  - Lucro: - # 1.620001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 99%|█████████▉| 278/281 [23:05<00:15,  5.09s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Vendeu:  $ 32.880001  - Lucro: $ 1.110001\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r 99%|█████████▉| 279/281 [23:10<00:10,  5.04s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 31.299999\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "\r100%|█████████▉| 280/281 [23:15<00:05,  5.08s/it]"
          ]
        },
        {
          "output_type": "stream",
          "name": "stdout",
          "text": [
            " - AI Trader Comprou:  $ 30.650000\n",
            "########################\n",
            "TOTAL PROFIT: 7.93499755859375\n",
            "########################\n"
          ]
        },
        {
          "output_type": "stream",
          "name": "stderr",
          "text": [
            "100%|██████████| 281/281 [23:20<00:00,  4.99s/it]\n"
          ]
        }
      ]
    },
    {
      "cell_type": "code",
      "source": [
        ""
      ],
      "metadata": {
        "id": "q3foDLCIMkC3"
      },
      "execution_count": None,

      "outputs": []
    }
  ],
  "metadata": {
    "colab": {
      "collapsed_sections": [],
      "name": "[SI] BITCOIN ROBOT 4.0 - Deep Q Learning - Gabriel e Vinicius",
      "provenance": []
    },
    "kernelspec": {
      "display_name": "Python 3",
      "name": "python3"
    },
    "language_info": {
      "name": "python"
    },
    "accelerator": "GPU"
  },
  "nbformat": 4,
  "nbformat_minor": 0
}
