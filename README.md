# 🗄️ Super Repositório – Desafios SQL 🚀  

Bem-vindo ao **super repositório de estudos SQL**!  
Aqui reuni todos os desafios práticos que venho desenvolvendo para **refinar habilidades em Banco de Dados**.  

📌 O foco está em **SQL Avançado**: índices, procedures, views, triggers, transações e até backup/recovery.  
Tudo documentado de forma simples, mas prática, como se fosse meu caderno de estudos.  

---

## 📂 Estrutura do Repositório  

📂 super-desafios-sql  
┣ 📜 README.md → Documentação do projeto  
┣ 📜 indices.sql → Criação de índices e queries otimizadas  
┣ 📜 procedures.sql → Procedures para universidade e e-commerce  
┣ 📜 views.sql → Criação de views e permissões  
┣ 📜 triggers.sql → Triggers de atualização e remoção (e-commerce)  
┣ 📜 transacoes.sql → Scripts de transações manuais  
┣ 📜 procedure_transacao.sql → Procedure com rollback e savepoint  
┣ 📜 ecommerce_backup.sql → Backup do banco (mysqldump)  

---

## 📌 PARTE 1 – Índices em Banco de Dados  

### 🎯 Objetivo  
Otimizar consultas criando índices somente nos campos **mais acessados e relevantes**, lembrando que:  

- Índices **aceleram consultas**.  
- Mas podem **pesar em inserts/updates**.  
- Só foram criados os **essenciais**.  

### 🔎 Queries & Índices  
```sql
-- Qual o departamento com maior número de pessoas?
CREATE INDEX idx_employee_dno ON employee(Dno);
SELECT Dno, COUNT(*) AS total
FROM employee
GROUP BY Dno
ORDER BY total DESC
LIMIT 1;

-- Quais são os departamentos por cidade?
CREATE INDEX idx_department_city ON department(Dlocation);
SELECT Dname, Dlocation
FROM department
ORDER BY Dlocation;

-- Relação de empregados por departamento
CREATE INDEX idx_employee_department ON employee(Dno);
SELECT e.Fname, e.Lname, d.Dname
FROM employee e
JOIN department d ON e.Dno = d.Dnumber
ORDER BY d.Dname;

````

# 📌 PARTE 2 – Procedures  

## 🎯 Objetivo  
Criar procedures para manipulação de dados em **universidade** e **e-commerce**, utilizando uma variável de controle para decidir a operação:  

- `1` → SELECT  
- `2` → UPDATE  
- `3` → DELETE  
- `4` → INSERT  

---

## 📝 Exemplo (Universidade)  

```sql
DELIMITER //
CREATE PROCEDURE manage_universidade (
    IN opcao INT,
    IN p_id INT,
    IN p_nome VARCHAR(100),
    IN p_endereco VARCHAR(150)
)
BEGIN
    CASE opcao
        WHEN 1 THEN 
            SELECT * FROM universidade WHERE id = p_id;
        WHEN 2 THEN 
            UPDATE universidade 
            SET nome = p_nome, endereco = p_endereco 
            WHERE id = p_id;
        WHEN 3 THEN 
            DELETE FROM universidade WHERE id = p_id;
        WHEN 4 THEN 
            INSERT INTO universidade (id, nome, endereco) 
            VALUES (p_id, p_nome, p_endereco);
    END CASE;
END //
DELIMITER ;
````
# 📌 PARTE 3 – Views e Permissões  

## 🎯 Objetivo  
Criar **views** para simplificar consultas e organizar permissões por tipo de usuário.  

---

## 🔎 Exemplos de Views  

```sql
-- Número de empregados por departamento e localidade
CREATE VIEW vw_emp_dept_loc AS
SELECT d.Dname, d.Dlocation, COUNT(e.Ssn) AS total_emp
FROM department d
JOIN employee e ON d.Dnumber = e.Dno
GROUP BY d.Dname, d.Dlocation;

-- Lista de departamentos e seus gerentes
CREATE VIEW vw_dept_gerente AS
SELECT d.Dname, e.Fname, e.Lname
FROM department d
JOIN employee e ON d.Mgr_ssn = e.Ssn;

-- Projetos com maior número de empregados
CREATE VIEW vw_proj_emp AS
SELECT p.Pname, COUNT(w.Essn) AS total_emp
FROM project p
JOIN works_on w ON p.Pnumber = w.Pno
GROUP BY p.Pname
ORDER BY total_emp DESC;
````

## 🔑 Permissões  

Para garantir segurança e organização do acesso às **views**, foram criados dois tipos de usuários:  

- **Gerente** → possui acesso a informações de empregados e departamentos.  
- **Employee** → possui acesso limitado, sem informações de gerentes.  

### 📜 Scripts  

```sql
-- Criando usuário gerente
CREATE USER 'gerente'@'localhost' IDENTIFIED BY '1234';
GRANT SELECT ON vw_emp_dept_loc TO 'gerente'@'localhost';
GRANT SELECT ON vw_dept_gerente TO 'gerente'@'localhost';

-- Criando usuário employee (sem acesso a info de gerentes)
CREATE USER 'employee'@'localhost' IDENTIFIED BY '1234';
GRANT SELECT ON vw_emp_dept_loc TO 'employee'@'localhost';

````

## 📌 PARTE 4 – Triggers no E-commerce  

### 🎯 Objetivo  
Criar **triggers** que atuam em momentos de `DELETE` e `INSERT/UPDATE` no cenário de **e-commerce**, garantindo histórico e consistência dos dados.  

---

### 🔎 Exemplos de Triggers  

#### 📌 Trigger para armazenar histórico antes de deletar cliente  
```sql
DELIMITER //
CREATE TRIGGER before_delete_cliente
BEFORE DELETE ON cliente
FOR EACH ROW
BEGIN
    INSERT INTO cliente_excluido (id, nome, email, data_exclusao)
    VALUES (OLD.id, OLD.nome, OLD.email, NOW());
END //
DELIMITER ;

````

## 📌 PARTE 5 – Transações  

### 🎯 Objetivo  
Garantir **atomicidade** nas operações do banco de dados utilizando:  
- `START TRANSACTION`  
- `COMMIT`  
- `ROLLBACK`  

---

### 🔎 Exemplo Simples  

```sql
SET autocommit = 0;

START TRANSACTION;

UPDATE cliente SET saldo = saldo - 500 WHERE id = 1;
UPDATE cliente SET saldo = saldo + 500 WHERE id = 2;

COMMIT;
-- ou em caso de erro
ROLLBACK;

````

### 📝 Transação com Procedure  

```sql
DELIMITER $$
CREATE PROCEDURE transferir_saldo(
    IN p_origem INT,
    IN p_destino INT,
    IN p_valor DECIMAL(10,2)
)
BEGIN
    -- Handler para capturar erros e executar rollback automático
    DECLARE exit handler FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
    END;

    -- Início da transação
    START TRANSACTION;

    -- Débito da conta de origem
    UPDATE cliente SET saldo = saldo - p_valor WHERE id = p_origem;

    -- Crédito na conta de destino
    UPDATE cliente SET saldo = saldo + p_valor WHERE id = p_destino;

    -- Confirma a transação
    COMMIT;
END$$
DELIMITER ;

````

## 📌 PARTE 6 – Backup & Recovery  

### 🎯 Objetivo
Realizar backup e recovery do banco `ecommerce` utilizando `mysqldump`.

### 🔎 Comandos

```bash
# Backup simples do banco ecommerce
mysqldump -u root -p ecommerce > ecommerce_backup.sql

# Restore do banco ecommerce
mysql -u root -p ecommerce < ecommerce_backup.sql

# Backup múltiplos bancos incluindo procedures e eventos
mysqldump -u root -p --databases ecommerce universidade --routines --events > multi_backup.sql

````

## 📚 Conclusão

Este super repositório é meu **guia prático de SQL avançado**, cobrindo:

✅ Índices para otimizar consultas  
✅ Procedures para manipulação de dados  
✅ Views e permissões para segurança e organização  
✅ Triggers para auditoria e consistência  
✅ Transações para atomicidade  
✅ Backup & Recovery para resiliência dos dados  

👨‍💻 Feito para estudos e para turbinar meu portfólio de **Engenharia de Dados & SQL** 🚀  

📌 Conecte-se comigo no LinkedIn: [Guilherme Felipe Cosmos](https://www.linkedin.com/in/guilherme-felipe-cosmos-8a4b1a183)


