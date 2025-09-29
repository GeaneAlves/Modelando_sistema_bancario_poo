from __future__ import annotations      # linha inserida pra evitar que referências circulares causem erros
from abc import ABC, abstractmethod   # Importa utilitários da classe ABC, neste caso, para criar a interface Transacao.
from datetime import datetime, date
import textwrap     # txtwrap.dedent() é um abiblioteca para formatar strings multilinha, imprimindo sem espaços desnecessários.


# =========================
#   Dominio (UML Classes)
# =========================

class Historico:
    """Mantém o histórico de transações da conta."""
    def __init__(self) -> None:
        self._transacoes: list[dict] = []

    @property
    def transacoes(self) -> list[dict]:
        return self._transacoes

    def adicionar_transacao(self, transacao: Transacao) -> None:
        self._transacoes.append(
            {
                "tipo": transacao.__class__.__name__,
                "valor": transacao.valor,
                "data": datetime.now().strftime("%d/%m/%Y %H:%M:%S"),
            }
        )


class Conta:
    """Conta genérica (superclasse)."""
    def __init__(self, numero: int, cliente: Cliente, agencia: str = "0001") -> None:
        self._saldo: float = 0.0
        self._numero: int = numero
        self._agencia: str = agencia
        self._cliente: Cliente = cliente
        self._historico: Historico = Historico()

    # --- propriedades ---
    @property
    def saldo(self) -> float:
        return self._saldo

    @property
    def numero(self) -> int:
        return self._numero

    @property
    def agencia(self) -> str:
        return self._agencia

    @property
    def cliente(self) -> Cliente:
        return self._cliente

    @property
    def historico(self) -> Historico:
        return self._historico

    #  operações de saque
    def sacar(self, valor: float) -> bool:
        """Saque 'padrão' da conta base (pode ser sobrescrito)."""
        if valor <= 0:
            print("\n@@@ Operação falhou! O valor informado é inválido. @@@")
            return False

        if valor > self._saldo:
            print("\n@@@ Operação falhou! Você não tem saldo suficiente. @@@")
            return False

        self._saldo -= valor
        print("\n=== Saque realizado com sucesso! ===")
        return True

    def depositar(self, valor: float) -> bool:
        if valor <= 0:
            print("\n@@@ Operação falhou! O valor informado é inválido. @@@")
            return False

        self._saldo += valor
        print("\n=== Depósito realizado com sucesso! ===")
        return True

    # fábrica para criar contas respeitando o UML
    @staticmethod
    def nova_conta(cliente: Cliente, numero: int) -> Conta:
        return Conta(numero=numero, cliente=cliente)


class ContaCorrente(Conta):
    """Especialização com limite por saque e limite diário de saques."""
    def __init__(self, numero: int, cliente: Cliente, agencia: str = "0001",
                 limite: float = 500.0, limite_saques: int = 3) -> None:
        super().__init__(numero, cliente, agencia)
        self.limite = limite
        self.limite_saques = limite_saques

    @staticmethod
    def nova_conta(cliente: Cliente, numero: int, agencia: str = "0001",
                   limite: float = 500.0, limite_saques: int = 3) -> ContaCorrente:
        return ContaCorrente(numero, cliente, agencia, limite, limite_saques)

    def _saques_do_dia(self) -> int:
        hoje = date.today().strftime("%d/%m/%Y")
        return sum(1 for t in self.historico.transacoes
                   if t["tipo"] == "Saque" and t["data"].startswith(hoje))

    def sacar(self, valor: float) -> bool:
        if valor <= 0:
            print("\n@@@ Operação falhou! O valor informado é inválido. @@@")
            return False

        if valor > self.saldo:
            print("\n@@@ Operação falhou! Você não tem saldo suficiente. @@@")
            return False

        if valor > self.limite:
            print("\n@@@ Operação falhou! O valor do saque excede o limite. @@@")
            return False

        if self._saques_do_dia() >= self.limite_saques:
            print("\n@@@ Operação falhou! Número máximo de saques diários excedido. @@@")
            return False

        self._saldo -= valor
        print("\n=== Saque realizado com sucesso! ===")
        return True


# ---------- Transações (interface + concretas) ----------

class Transacao(ABC):
    def __init__(self, valor: float) -> None:
        self._valor = valor

    @property
    def valor(self) -> float:
        return self._valor

    @abstractmethod
    def registrar(self, conta: Conta) -> None:
        """Executa a transação na conta e grava no histórico."""


class Deposito(Transacao):
    def registrar(self, conta: Conta) -> None:
        sucesso = conta.depositar(self.valor)
        if sucesso:
            conta.historico.adicionar_transacao(self)


class Saque(Transacao):
    def registrar(self, conta: Conta) -> None:
        sucesso = conta.sacar(self.valor)
        if sucesso:
            conta.historico.adicionar_transacao(self)


# ---------- Clientes ----------

class Cliente:
    def __init__(self, endereco: str) -> None:
        self.endereco = endereco
        self.contas: list[Conta] = []

    def realizar_transacao(self, conta: Conta, transacao: Transacao) -> None:
        transacao.registrar(conta)

    def adicionar_conta(self, conta: Conta) -> None:
        self.contas.append(conta)


class PessoaFisica(Cliente):
    def __init__(self, nome: str, data_nascimento: str, cpf: str, endereco: str) -> None:
        super().__init__(endereco)
        self.cpf = cpf
        self.nome = nome
        self.data_nascimento = data_nascimento


# =========================
#     Funções utilitárias
# =========================

def menu():
    menu = """\n
    ================ MENU ================
    [d]\tDepositar
    [s]\tSacar
    [e]\tExtrato
    [nc]\tNova conta
    [lc]\tListar contas
    [nu]\tNovo usuário
    [q]\tSair
    => """
    return input(textwrap.dedent(menu))


def filtrar_cliente(cpf: str, clientes: list[PessoaFisica]) -> PessoaFisica | None:
    for cliente in clientes:
        if isinstance(cliente, PessoaFisica) and cliente.cpf == cpf:
            return cliente
    return None


def recuperar_conta_por_numero(cliente: Cliente, numero_conta: int) -> Conta | None:
    for conta in cliente.contas:
        if conta.numero == numero_conta:
            return conta
    return None

# exibição do extrato

def exibir_extrato(conta: Conta):
    print("\n================ EXTRATO ================")
    transacoes = conta.historico.transacoes
    if not transacoes:
        print("Não foram realizadas movimentações.")
    else:
        for t in transacoes:
            print(f"{t['tipo']}:\tR$ {t['valor']:.2f}\t({t['data']})")
    print(f"\nSaldo:\t\tR$ {conta.saldo:.2f}")
    print("==========================================")

# criando um novo usuário

def criar_usuario(clientes: list[PessoaFisica]):
    cpf = input("Informe o CPF (somente número): ")
    if filtrar_cliente(cpf, clientes):
        print("\n@@@ Já existe usuário com esse CPF! @@@")
        return

    nome = input("Informe o nome completo: ")
    data_nascimento = input("Informe a data de nascimento (dd-mm-aaaa): ")
    endereco = input("Informe o endereço (logradouro, nro - bairro - cidade/sigla estado): ")

    cliente = PessoaFisica(nome=nome, data_nascimento=data_nascimento, cpf=cpf, endereco=endereco)
    clientes.append(cliente)
    print("=== Usuário criado com sucesso! ===")

# criando uma nova conta

def criar_conta(contas: list[Conta], clientes: list[PessoaFisica], agencia_padrao="0001",
                limite=500.0, limite_saques=3):
    cpf = input("Informe o CPF do usuário: ")
    cliente = filtrar_cliente(cpf, clientes)

    if not cliente:
        print("\n@@@ Usuário não encontrado, fluxo de criação de conta encerrado! @@@")
        return

    numero_conta = len(contas) + 1
    conta = ContaCorrente.nova_conta(
        cliente=cliente,
        numero=numero_conta,
        agencia=agencia_padrao,
        limite=limite,
        limite_saques=limite_saques,
    )
    cliente.adicionar_conta(conta)
    contas.append(conta)
    print("\n=== Conta criada com sucesso! ===")
    print(f"Agência: {conta.agencia} | C/C: {conta.numero} | Titular: {cliente.nome}")


def listar_contas(contas: list[Conta]):
    if not contas:
        print("\n@@@ Não há contas cadastradas. @@@")
        return

    for conta in contas:
        linha = f"""\
            Agência:\t{conta.agencia}
            C/C:\t\t{conta.numero}
            Titular:\t{conta.cliente.nome}
        """
        print("=" * 100)
        print(textwrap.dedent(linha))


# =========================
#          App
# =========================

def main():
    AGENCIA = "0001"
    LIMITE = 500.0
    LIMITE_SAQUES = 3

    clientes: list[PessoaFisica] = []
    contas: list[Conta] = []

    while True:
        opcao = menu()

        if opcao == "d":
            cpf = input("Informe o CPF do titular: ")
            cliente = filtrar_cliente(cpf, clientes)
            if not cliente:
                print("\n@@@ Cliente não encontrado! @@@")
                continue

            try:
                numero = int(input("Informe o número da conta: "))
            except ValueError:
                print("\n@@@ Número de conta inválido. @@@")
                continue

            conta = recuperar_conta_por_numero(cliente, numero)
            if not conta:
                print("\n@@@ Conta não encontrada para este cliente. @@@")
                continue

            try:
                valor = float(input("Informe o valor do depósito: "))
            except ValueError:
                print("\n@@@ Valor inválido. @@@")
                continue

            transacao = Deposito(valor)
            cliente.realizar_transacao(conta, transacao)

        elif opcao == "s":
            cpf = input("Informe o CPF do titular: ")
            cliente = filtrar_cliente(cpf, clientes)
            if not cliente:
                print("\n@@@ Cliente não encontrado! @@@")
                continue

            try:
                numero = int(input("Informe o número da conta: "))
            except ValueError:
                print("\n@@@ Número de conta inválido. @@@")
                continue

            conta = recuperar_conta_por_numero(cliente, numero)
            if not conta:
                print("\n@@@ Conta não encontrada para este cliente. @@@")
                continue

            try:
                valor = float(input("Informe o valor do saque: "))
            except ValueError:
                print("\n@@@ Valor inválido. @@@")
                continue

            # Para garantir os limites da ContaCorrente, eles já são validados em ContaCorrente.sacar()
            transacao = Saque(valor)
            cliente.realizar_transacao(conta, transacao)

        elif opcao == "e":
            if not contas:
                print("\n@@@ Não há contas cadastradas. @@@")
                continue

            try:
                numero = int(input("Informe o número da conta para extrato: "))
            except ValueError:
                print("\n@@@ Número de conta inválido. @@@")
                continue

            # encontrando a conta pelo número
            conta = next((c for c in contas if c.numero == numero), None)
            if not conta:
                print("\n@@@ Conta não encontrada. @@@")
                continue

            exibir_extrato(conta)

        elif opcao == "nu":
            criar_usuario(clientes)

        elif opcao == "nc":
            criar_conta(contas, clientes, agencia_padrao=AGENCIA,
                        limite=LIMITE, limite_saques=LIMITE_SAQUES)

        elif opcao == "lc":
            listar_contas(contas)

        elif opcao == "q":
            break

        else:
            print("Operação inválida, por favor selecione novamente a operação desejada.")


if __name__ == "__main__":
    main()

