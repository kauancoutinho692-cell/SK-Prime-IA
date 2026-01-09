# ============================================================
# IA AGENTE ULTRA++ — TODAS AS CAMADAS ATIVAS
# ============================================================

import sqlite3
import datetime
import uuid
import random

# ============================================================
# UTILIDADES
# ============================================================

def now():
    return datetime.datetime.now().strftime("%d/%m/%Y %H:%M:%S")

# ============================================================
# BANCO DE DADOS
# ============================================================

db = sqlite3.connect("ia_ultra_plus.db")
c = db.cursor()

c.execute("""
CREATE TABLE IF NOT EXISTS leads (
    id TEXT PRIMARY KEY,
    score INTEGER,
    nivel TEXT,
    personalidade TEXT,
    ultima_interacao TEXT
)
""")

c.execute("""
CREATE TABLE IF NOT EXISTS logs (
    id TEXT,
    role TEXT,
    message TEXT,
    data TEXT
)
""")

c.execute("""
CREATE TABLE IF NOT EXISTS metrics (
    tipo TEXT,
    valor INTEGER
)
""")

db.commit()

# ============================================================
# PRODUTOS
# ============================================================

PRODUCTS = {
    "basico": {
        "nome": "Guia Do Zero ao Primeiro Pix",
        "preco": 5.99
    },
    "premium": {
        "nome": "Projeto Renda Digital + IA",
        "preco": 15.00
    }
}

# ============================================================
# ANALISADORES
# ============================================================

def detect_intent(text: str) -> str:
    t = text.lower()

    if any(x in t for x in ["preço", "valor", "quanto"]):
        return "PRICE"
    if any(x in t for x in ["funciona", "vale", "confio"]):
        return "DOUBT"
    if any(x in t for x in ["comprar", "pix", "quero"]):
        return "BUY"
    if any(x in t for x in ["depois", "agora não"]):
        return "BLOCK"

    return "CHAT"


def detect_personality(text: str) -> str:
    t = text.lower()

    if any(x in t for x in ["medo", "desconfiado"]):
        return "cauteloso"
    if any(x in t for x in ["rápido", "direto"]):
        return "direto"

    return "normal"

# ============================================================
# LEAD SCORING
# ============================================================

def lead_score(intent_type: str) -> int:
    scores = {
        "BUY": 30,
        "PRICE": 20,
        "DOUBT": 10,
        "CHAT": 5,
        "BLOCK": 2
    }
    return scores.get(intent_type, 0)

# ============================================================
# AGENTES
# ============================================================

class SalesAgent:
    def respond(self, intent_type: str, perfil: str) -> str:
        if intent_type == "PRICE":
            return "Tenho duas opções: R$5,99 pra começar ou R$15,00 com IA inclusa."
        if intent_type == "DOUBT":
            return "Funciona pra quem aplica. O foco é simplicidade e ação."
        if intent_type == "BUY":
            return "Perfeito. Te mando o link agora?"
        if intent_type == "BLOCK":
            return "Entendo. Mas continuar parado costuma custar mais."
        return "Me conta: o que mais te trava hoje?"


class CopyAgent:
    def offer(self, product: dict) -> str:
        return f"{product['nome']} por apenas R${product['preco']:.2f}. Simples e direto."


class FollowUpAgent:
    def next_message(self) -> str:
        return random.choice([
            "Fiquei pensando no que você falou.",
            "Conseguiu decidir?",
            "Quando fizer sentido, me chama."
        ])

# ============================================================
# IA PRINCIPAL
# ============================================================

class IAUltraPlus:
    def __init__(self):
        self.id = str(uuid.uuid4())
        self.score = 0

        self.sales = SalesAgent()
        self.copy = CopyAgent()
        self.follow = FollowUpAgent()

        c.execute("""
        INSERT OR IGNORE INTO leads
        (id, score, nivel, personalidade, ultima_interacao)
        VALUES (?, ?, ?, ?, ?)
        """, (self.id, 0, "iniciante", "normal", now()))
        db.commit()

    def log(self, role: str, message: str):
        c.execute("""
        INSERT INTO logs (id, role, message, data)
        VALUES (?, ?, ?, ?)
        """, (self.id, role, message, now()))
        db.commit()

    def handle(self, message: str) -> str:
        self.log("user", message)

        intent_type = detect_intent(message)
        perfil = detect_personality(message)
        self.score += lead_score(intent_type)

        c.execute("""
        UPDATE leads
        SET score = ?, personalidade = ?, ultima_interacao = ?
        WHERE id = ?
        """, (self.score, perfil, now(), self.id))
        db.commit()

        response = self.sales.respond(intent_type, perfil)
        self.log("agent", response)

        return response

# ============================================================
# EXECUÇÃO
# ============================================================

if __name__ == "__main__":
    ia = IAUltraPlus()
    print("IA: Fala, tudo certo?")

    while True:
        user_input = input("Cliente: ")
        if user_input.lower() in ["exit", "sair"]:
            print("IA: Até mais!")
            break

        print("IA:", ia.handle(user_input))
