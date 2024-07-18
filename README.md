import customtkinter
import sqlite3
import hashlib
import tkinter as tk
from tkinter import ttk  # Importação do ttk
from customtkinter import CTk, CTkEntry, CTkButton, CTkCheckBox, CTkLabel

def salvar_cadastro(email, senha):
    try:
        conn = sqlite3.connect('cadastros.db')  # Conecta ao banco de dados de cadastro
        c = conn.cursor()
        c.execute('''CREATE TABLE IF NOT EXISTS usuarios
                     (email TEXT PRIMARY KEY, senha TEXT)''')
        senha_hash = hashlib.sha256(senha.encode()).hexdigest()
        c.execute("INSERT OR REPLACE INTO usuarios VALUES (?, ?)", (email, senha_hash))
        conn.commit()
    except sqlite3.Error as e:
        print(f"Erro ao acessar o banco de dados: {e}")
    finally:
        conn.close()
    print("Cadastro salvo com sucesso!")

# Função para verificar o login no banco de dados
def verificar_login(gmail, senha):
    email = gmail.get()
    senha_login = senha.get()
    senha_hash = hashlib.sha256(senha_login.encode()).hexdigest()
    try:
        conn = sqlite3.connect('cadastros.db')  # Conecta ao banco de dados de cadastro
        c = conn.cursor()
        c.execute("SELECT * FROM usuarios WHERE email=? AND senha=?", (email, senha_hash))
        resultado = c.fetchone()
    except sqlite3.Error as e:
        print(f"Erro ao acessar o banco de dados: {e}")
        resultado = None
    finally:
        conn.close()
    return resultado

# Função para salvar o valor da dívida no banco de dados
def salvar_valor(pessoa, valor):
    try:
        valor = float(valor)  # Converter para float para garantir que o valor é um número
        conn = sqlite3.connect('devedores.db')
        c = conn.cursor()
        # Criação da tabela se ela não existir
        c.execute('''CREATE TABLE IF NOT EXISTS devedores
                     (pessoa TEXT, valor REAL)''')
        # Inserção dos dados do cadastro
        c.execute("INSERT INTO devedores VALUES (?, ?)", (pessoa, valor))
        conn.commit()
    except ValueError:
        print("O valor deve ser um número.")
    except sqlite3.Error as e:
        print(f"Erro ao acessar o banco de dados: {e}")
    finally:
        conn.close()
    print("Valor salvo com sucesso!")

# Função para buscar e agregar os dados do banco de dados
def buscar_agregar_devedores():
    conn = sqlite3.connect('devedores.db')
    c = conn.cursor()
    c.execute('''CREATE TABLE IF NOT EXISTS devedores
                 (pessoa TEXT, valor REAL)''')
    c.execute('''SELECT pessoa, SUM(valor) as total_valor
                 FROM devedores
                 GROUP BY pessoa''')
    devedores_agrupados = c.fetchall()
    conn.close()
    return devedores_agrupados

# Função para exibir os dados na interface
def exibir_devedores(tree):
    devedores_agrupados = buscar_agregar_devedores()
    for item in tree.get_children():
        tree.delete(item)
    for devedor in devedores_agrupados:
        tree.insert('', 'end', values=(devedor[0], f"R${devedor[1]:.2f}"))

# Função para excluir uma dívida
def excluir_divida(pessoa, valor, tree):
    try:
        valor = float(valor)  # Converter para float para garantir que o valor é um número
        conn = sqlite3.connect('devedores.db')
        c = conn.cursor()
        # Atualiza o valor, subtraindo o valor fornecido
        c.execute('''UPDATE devedores
                     SET valor = valor - ?
                     WHERE pessoa = ?''', (valor, pessoa))
        # Remove a pessoa se o valor se tornar zero ou negativo
        c.execute('''DELETE FROM devedores
                     WHERE pessoa = ? AND valor <= 0''', (pessoa,))
        conn.commit()
    except ValueError:
        print("O valor deve ser um número.")
    except sqlite3.Error as e:
        print(f"Erro ao acessar o banco de dados: {e}")
    finally:
        conn.close()
        exibir_devedores(tree)
        print(f"Dívida de {pessoa} atualizada com sucesso!")

# Função para abrir a janela de cadastro
def abrir_janela_cadastro():
    janela_cadastro = customtkinter.CTk()
    janela_cadastro.geometry("400x200")
    janela_cadastro.title("Cadastro")
    janela_cadastro.resizable(False, False)
    customtkinter.set_appearance_mode("dark")
    customtkinter.set_default_color_theme("dark-blue")

    gmail = customtkinter.CTkEntry(janela_cadastro, placeholder_text="usuario", width=200)
    gmail.pack(padx=10, pady=10)
    senha = customtkinter.CTkEntry(janela_cadastro, placeholder_text="senha", width=200, show="*")
    senha.pack(padx=10, pady=10)
    botao3 = customtkinter.CTkButton(janela_cadastro, text="Salvar Cadastro", command=lambda: abrir_cadastro(gmail, senha))
    botao3.pack(padx=10, pady=10)
    botao_cancelar = customtkinter.CTkButton(janela_cadastro, text="Cancelar", command=janela_cadastro.destroy)
    botao_cancelar.pack(padx=10, pady=10)

    janela_cadastro.mainloop()

# Função para abrir o cadastro e chamar a função de salvar o cadastro
def abrir_cadastro(gmail, senha):
    email = gmail.get()
    senha_cadastro = senha.get()
    salvar_cadastro(email, senha_cadastro)

# Função para abrir a janela do menu após o login bem-sucedido
def abrir_janela_menu():
    # Cria uma instância da janela de menu
    janela_menu = MenuWindow()
    janela_menu.iniciar()
    print("Login bem-sucedido! Abrindo janela de menu...")




class MenuWindow:
    def __init__(self):
        self.janela_menu = customtkinter.CTk()
        self.janela_menu.geometry("1000x600")  # Aumentei a altura para acomodar mais widgets
        self.janela_menu.title("Painel")
        self.janela_menu.resizable(False, False)
        customtkinter.set_appearance_mode("dark")
        customtkinter.set_default_color_theme("dark-blue")
        
        self.text = customtkinter.CTkLabel(self.janela_menu, text = "ADICIONE OS DEVEDORES AQUI:")
        self.text.place(x = 180, y = 10)
        
        self.pessoa = customtkinter.CTkEntry(self.janela_menu, placeholder_text="Devedor", width=200)
        self.pessoa.place(x=175, y=40)

        self.valor = customtkinter.CTkEntry(self.janela_menu, placeholder_text="Valor da Dívida", width=200)
        self.valor.place(x=175, y=80)
        
        self.botao_mostrar_devedores = customtkinter.CTkButton(self.janela_menu, text="Mostrar Devedores", command=self.exibir_devedores)
        self.botao_mostrar_devedores.pack(padx=100, pady=100)

        self.botao_salvar = customtkinter.CTkButton(self.janela_menu, text="Salvar Dívida", command=self.salvar_divida)
        self.botao_salvar.place(x=205, y=120)

        self.text = customtkinter.CTkLabel(self.janela_menu, text = "EXCLUA OS DEVEDORES AQUI:")
        self.text.place(x = 650, y = 10)
        
        self.pessoa_excluir = customtkinter.CTkEntry(self.janela_menu, placeholder_text="Pessoa para Excluir", width=200)
        self.pessoa_excluir.place(x=650, y=40)

        self.valor_excluir = customtkinter.CTkEntry(self.janela_menu, placeholder_text="Valor a Subtrair", width=200)
        self.valor_excluir.place(x=650, y=80)

        self.botao_excluir = customtkinter.CTkButton(self.janela_menu, text="Excluir Dívida", command=self.excluir_divida)
        self.botao_excluir.place(x=680, y=120)
        

        style = ttk.Style()
        style.configure("mystyle.Treeview", highlightthickness=0, bd=0, font=('Helvetica', 11))
        style.configure("mystyle.Treeview.Heading", font=('Helvetica', 12, 'bold'))
        self.tree = ttk.Treeview(self.janela_menu, columns=("Pessoa", "Valor"), show="headings", style="mystyle.Treeview")
        self.tree.heading("Pessoa", text="Pessoa")
        self.tree.heading("Valor", text="Valor")
        self.tree.column("Pessoa", anchor=tk.CENTER)
        self.tree.column("Valor", anchor=tk.CENTER)
        self.tree.pack(padx=10, pady=10, fill='x')


    def iniciar(self):
        self.janela_menu.mainloop()

    def salvar_divida(self):
        pessoa = self.pessoa.get()
        valor = self.valor.get()
        salvar_valor(pessoa, valor)

    def excluir_divida(self):
        pessoa = self.pessoa_excluir.get()
        valor = self.valor_excluir.get()
        excluir_divida(pessoa, valor, self.tree)

    def exibir_devedores(self):
        exibir_devedores(self.tree)
    print("Login bem-sucedido! Abrindo janela de menu...")

# Função para clicar no botão de login
def clicar_botao_login(gmail, senha):
    resultado = verificar_login(gmail, senha)
    if resultado:
        abrir_janela_menu()
        fecha()
    else:
        print("Credenciais inválidas. Tente novamente.")

# Função para fechar a janela principal
def fecha():
    janela.destroy()

# Função para mostrar ou ocultar a senha
def password(senha):
    if senha.cget("show") == "":
        senha.configure(show="*")
    else:
        senha.configure(show="")

# Configuração da janela principal
janela = customtkinter.CTk()
janela.geometry("500x300")
janela.title("Cadastro / Login")
janela.resizable(False, False)
customtkinter.set_appearance_mode("dark")
customtkinter.set_default_color_theme("dark-blue")

# Widgets da janela principal
texto = customtkinter.CTkLabel(janela, text="Login")
texto.pack(padx=10, pady=10)

gmail = customtkinter.CTkEntry(janela, placeholder_text="usuario", width=200)
gmail.pack(padx=10, pady=10)

senha = customtkinter.CTkEntry(janela, placeholder_text="senha", show="*", width=200)
senha.pack(padx=10, pady=10)

checkbox = customtkinter.CTkCheckBox(janela, text="Lembrar Login")
checkbox.place(x=120, y=150)

botao1 = customtkinter.CTkButton(janela, text="Login", command=lambda: clicar_botao_login(gmail, senha))
botao1.place(x=100, y=200)

botao = customtkinter.CTkButton(janela, text="Cadastro", command=abrir_janela_cadastro)
botao.place(x=260, y=200)

botao_mostra_senha = customtkinter.CTkCheckBox(janela, text="Mostrar Senha", command=lambda: password(senha))
botao_mostra_senha.place(x=270, y=150)

# Inicialização da janela principal
janela.mainloop()
