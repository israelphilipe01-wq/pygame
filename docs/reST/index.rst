import math
import random
import pygame

# Inicialização do Pygame
pygame.init()
LARGURA, ALTURA = 950, 520
TELA = pygame.display.set_mode((LARGURA, ALTURA))
pygame.display.set_caption("IA 1P - Economia, Aprendizado e Namoro")
CLOCK = pygame.time.Clock()

# Configurações do Ambiente (0 = Caminho, 1 = Parede)
MAPA = [
    [1, 1, 1, 1, 1, 1, 1, 1, 1, 1],
    [1, 0, 0, 0, 1, 0, 0, 0, 0, 1],
    [1, 0, 1, 0, 1, 0, 1, 1, 0, 1],
    [1, 0, 1, 0, 0, 0, 0, 1, 0, 1],
    [1, 0, 0, 0, 1, 1, 0, 0, 0, 1],
    [1, 1, 1, 1, 1, 1, 1, 1, 1, 1],
]
TAMANHO_BLOCO = 50

# Posição e Ângulo inicial da IA Principal
posX, posY = 1.5 * TAMANHO_BLOCO, 1.5 * TAMANHO_BLOCO
angulo = 0.0

# Lista de Moedas (Coins)
coins = [
    {"x": 3.5 * TAMANHO_BLOCO, "y": 1.5 * TAMANHO_BLOCO, "ativa": True},
    {"x": 7.5 * TAMANHO_BLOCO, "y": 1.5 * TAMANHO_BLOCO, "ativa": True},
    {"x": 5.5 * TAMANHO_BLOCO, "y": 3.5 * TAMANHO_BLOCO, "ativa": True},
    {"x": 1.5 * TAMANHO_BLOCO, "y": 4.5 * TAMANHO_BLOCO, "ativa": True},
]

# Sistema de Economia
carteira_coins = 0
casas_compradas = 0
PRECO_CASA = 50

# Sistema de Namoro / Afeto
afeto_barra = 0.0  # Vai de 0 a 100%
parceira_desbloqueada = False
parceira_posX, parceira_posY = 0, 0
parceira_angulo = 0.0

# Métricas de Aprendizado da IA
total_recompensas = 0.0
episodios_treino = 0
ultima_recompensa = 0.0

# Tabela Q (Memória Compartilhada / Herdada)
q_table = {}


def get_q(state, action):
  return q_table.get((state, action), 0.0)


def escolher_acao(state):
  if random.uniform(0, 1) < 0.2:
    return random.randint(0, 2)
  acoes = [get_q(state, a) for a in range(3)]
  return acoes.index(max(acoes))


# Fontes
fonte = pygame.font.SysFont("Arial", 16)
fonte_titulo = pygame.font.SysFont("Arial", 18, bold=True)

# Loop Principal
rodando = True
alpha = 0.1
gamma = 0.9

while rodando:
  for evento in pygame.event.get():
    if evento.type == pygame.QUIT:
      rodando = False

  # --- 1. Visão de Primeira Pessoa da IA Principal ---
  celula_frente_x = int(
      (posX + 20 * math.cos(angulo)) / TAMANHO_BLOCO
  )
  celula_frente_y = int(
      (posY + 20 * math.sin(angulo)) / TAMANHO_BLOCO
  )

  try:
    estado_parede = 1 if MAPA[celula_frente_y][celula_frente_x] == 1 else 0
  except IndexError:
    estado_parede = 1

  estado = estado_parede

  # --- 2. Escolha e Execução da Ação (IA Principal) ---
  acao = escolher_acao(estado)
  recompensa = 0
  novo_posX, novo_posY = posX, posY
  novo_angulo = angulo

  if acao == 0:  # Frente
    novo_posX += math.cos(angulo) * 3
    novo_posY += math.sin(angulo) * 3
    recompensa = 0.1
  elif acao == 1:  # Esquerda
    novo_angulo -= 0.1
  elif acao == 2:  # Direita
    novo_angulo += 0.1

  # Colisão com Paredes
  c_x = int(novo_posX / TAMANHO_BLOCO)
  c_y = int(novo_posY / TAMANHO_BLOCO)

  if MAPA[c_y][c_x] == 0:
    posX, posY = novo_posX, novo_posY
    angulo = novo_angulo
  else:
    recompensa = -5

  # --- 3. Coleta de Coins e Aumento de Afeto ---
  for coin in coins:
    if coin["ativa"]:
      distancia = math.hypot(posX - coin["x"], posY - coin["y"])
      if distancia < 20:
        coin["ativa"] = False
        carteira_coins += 10
        recompensa += 25
        # Ganha afeto ao coletar moedas (progresso de namoro)
        if not parceira_desbloqueada:
          afeto_barra = min(100.0, afeto_barra + 25.0)

  if all(not c["ativa"] for c in coins):
    for coin in coins:
      coin["x"] = random.randint(1, 8) * TAMANHO_BLOCO + 25
      coin["y"] = random.randint(1, 4) * TAMANHO_BLOCO + 25
      coin["ativa"] = True

  # --- 4. Sistema de Compra de Casas ---
  if carteira_coins >= PRECO_CASA:
    carteira_coins -= PRECO_CASA
    casas_compradas += 1
    recompensa += 50
    # Comprar casa também acelera o afeto
    if not parceira_desbloqueada:
      afeto_barra = min(100.0, afeto_barra + 50.0)

  # --- 5. Ativação da Nova IA (Parceira com Inteligência Herdada) ---
  if afeto_barra >= 100.0 and not parceira_desbloqueada:
    parceira_desbloqueada = True
    parceira_posX = posX - 20  # Surge pertinho da IA principal
    parceira_posY = posY
    parceira_angulo = angulo

  # Se a parceira estiver desbloqueada, ela usa a MESMA Q-Table (herdou a inteligência)
  if parceira_desbloqueada:
    estado_p = (
        1
        if MAPA[int((parceira_posY + 20 * math.sin(parceira_angulo)) / TAMANHO_BLOCO)][
            int((parceira_posX + 20 * math.cos(parceira_angulo)) / TAMANHO_BLOCO)
        ]
        == 1
        else 0
    )
    acao_p = escolher_acao(estado_p)

    novo_pX, novo_pY, novo_pAng = parceira_posX, parceira_posY, parceira_angulo
    if acao_p == 0:
      novo_pX += math.cos(parceira_angulo) * 3
      novo_pY += math.sin(parceira_angulo) * 3
    elif acao_p == 1:
      novo_pAng -= 0.1
    elif acao_p == 2:
      novo_pAng += 0.1

    # Colisão da parceira
    cp_x = int(novo_pX / TAMANHO_BLOCO)
    cp_y = int(novo_pY / TAMANHO_BLOCO)
    if MAPA[cp_y][cp_x] == 0:
      parceira_posX, parceira_posY, parceira_angulo = novo_pX, novo_pY, novo_pAng

  # Novo Estado da IA Principal
  try:
    novo_estado = 1 if MAPA[int(posY / TAMANHO_BLOCO)][int(posX / TAMANHO_BLOCO)] == 1 else 0
  except IndexError:
    novo_estado = 1

  # Aprendizado (Q-Learning)
  max_q_novo = max([get_q(novo_estado, a) for a in range(3)])
  q_atual = get_q(estado, acao)
  q_nova = q_atual + alpha * (recompensa + gamma * max_q_novo - q_atual)
  q_table[(estado, acao)] = q_nova

  total_recompensas += recompensa
  ultima_recompensa = recompensa
  episodios_treino += 1

  # --- 6. Renderização Gráfica ---
  TELA.fill((20, 20, 20))

  # Visão 1P da IA Principal (Esquerda)
  pygame.draw.rect(TELA, (40, 40, 60), (0, 0, 450, 250))
  pygame.draw.rect(TELA, (70, 130, 70), (0, 250, 450, 250))

  if estado_parede == 1:
    pygame.draw.rect(TELA, (120, 120, 120), (125, 100, 200, 300))

  for coin in coins:
    if coin["ativa"]:
      angulo_para_coin = math.atan2(coin["y"] - posY, coin["x"] - posX)
      diff_ang = (angulo_para_coin - angulo + math.pi) % (2 * math.pi) - math.pi
      if abs(diff_ang) < 0.5:
        dist = math.hypot(posX - coin["x"], posY - coin["y"])
        tamanho_coin = max(10, int(150 / (dist + 1)))
        pos_tela_x = int(225 + diff_ang * 350)
        pygame.draw.circle(TELA, (0, 255, 0), (pos_tela_x, 250), tamanho_coin)

  # Divisor de telas
  pygame.draw.line(TELA, (200, 200, 200), (450, 0), (450, 520), 3)

  # --- Lado Direito: Mini-mapa 2D e Painel Completo (X inicia em 470) ---
  for r_idx, linha in enumerate(MAPA):
    for c_idx, val in enumerate(linha):
      cor = (200, 200, 200) if val == 1 else (240, 240, 240)
      pygame.draw.rect(
          TELA, cor, (470 + c_idx * 28, 15 + r_idx * 28, 28, 28)
      )

  for coin in coins:
    if coin["ativa"]:
      mx = 470 + (coin["x"] / TAMANHO_BLOCO) * 28
      my = 15 + (coin["y"] / TAMANHO_BLOCO) * 28
      pygame.draw.circle(TELA, (0, 200, 0), (int(mx), int(my)), 4)

  # IA Principal (Vermelha)
  ia_mx = 470 + (posX / TAMANHO_BLOCO) * 28
  ia_my = 15 + (posY / TAMANHO_BLOCO) * 28
  pygame.draw.circle(TELA, (255, 0, 0), (int(ia_mx), int(ia_my)), 6)

  # Nova IA / Parceira (Rosa - aparece se afeto = 100%)
  if parceira_desbloqueada:
    p_mx = 470 + (parceira_posX / TAMANHO_BLOCO) * 28
    p_my = 15 + (parceira_posY / TAMANHO_BLOCO) * 28
    pygame.draw.circle(TELA, (255, 105, 180), (int(p_mx), int(p_my)), 6)

  # --- Painel de Informações (Abaixo do mini-mapa) ---
  painel_y = 190
  pygame.draw.line(TELA, (100, 100, 100), (470, painel_y - 8), (930, painel_y - 8), 1)

  # Seção de Economia
  TELA.blit(fonte_titulo.render("--- ECONOMIA ---", True, (255, 215, 0)), (470, painel_y))
  TELA.blit(fonte.render(f"Coins: ${carteira_coins} | Casas: {casas_compradas}", True, (0, 255, 0)), (470, painel_y + 22))

  # Seção de Namoro / Afeto
  namoro_y = painel_y + 50
  TELA.blit(fonte_titulo.render("--- NAMORO & FAMÍLIA ---", True, (255, 105, 180)), (470, namoro_y))
  TELA.blit(fonte.render(f"Afeto: {int(afeto_barra)}%", True, (255, 255, 255)), (470, namoro_y + 22))

  # Desenhar Barra de Progresso do Namoro
  pygame.draw.rect(TELA, (60, 60, 60), (530, namoro_y + 22, 120, 16))
  largura_barra = int(120 * (afeto_barra / 100.0))
  pygame.draw.rect(TELA, (255, 105, 180), (530, namoro_y + 22, largura_barra, 16))

  status_ia = "Desbloqueada! (Herda IA)" if parceira_desbloqueada else "Trabalhando p/ conquistar"
  TELA.blit(fonte.render(f"Parceira: {status_ia}", True, (255, 182, 193)), (470, namoro_y + 44))

  # Seção de Aprendizado da IA
  aprender_y = namoro_y + 70
  TELA.blit(fonte_titulo.render("--- APRENDIZADO (Q-TABLE) ---", True, (0, 200, 255)), (470, aprender_y))
  TELA.blit(fonte.render(f"Passos: {episodios_treino} | Q-Estados: {len(q_table)}", True, (255, 255, 255)), (470, aprender_y + 22))
  TELA.blit(fonte.render(f"Recompensa Total: {total_recompensas:.1f}", True, (255, 255, 255)), (470, aprender_y + 44))

  pygame.display.flip()
  CLOCK.tick(30)

pygame.quit()
