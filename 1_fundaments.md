# DDD Foundaments

Estratégia que pode ajudar estruturar melhor os domínios para garantir foco no negócio,<br />
tem como objetivo a facilitar quando o software é muito completo de trabalhar.<br />

O código pretende refletir corretamente as regras do domínio, onde o vocabulário de cada participante está estruturada e alinhada com negócio.<br />
Isso impede que o desenvolvedor tome como verdade definições, nomeclaturas e até decisões no código sem estar alinhado com a documentação do DDD.<br />


# Entities

Entidades, é um objeto do domínio que possui identidade própria e contínua ao longo do tempo, mesmo que seus atributos mudem.

```
type Pedido struct {
    ID         string       // ← identidade única
    ClienteID  string
    Itens      []ItemPedido
    Status     string
    CriadoEm   time.Time
}

func (p *Pedido) AdicionarItem(item ItemPedido) {
    p.Itens = append(p.Itens, item)
}
```

#### Quando criar Entidades

Quando o objeto precisa ser rastreado individualmente.<br />
Quando você não pode substituir um objeto por outro idêntico sem mudar o contexto.


# Value Objects

No DDD (Domain-Driven Design), um Value Object (ou Objeto de Valor) representa um conceito do domínio que não tem identidade própria.<br />
Ele é imutável, comparado por seus atributos e não existe sozinho, mas sempre está associado a uma Entity ou outro contexto.

```
type Endereco struct {
    Rua     string
    Cidade  string
    Estado  string
    CEP     string
}
```


# Services


Service (ou Serviço de Domínio) é um componente que encapsula uma lógica de negócio importante que não pertence diretamente a uma Entity ou Value Object.
Um Service é uma classe ou função que executa ações no domínio, geralmente envolvendo várias entidades ou operações que não se encaixam bem em um único objeto.

```
type Conta struct {
    Numero string
    Saldo  float64
}

func (c *Conta) Debitar(valor float64) error {
    if c.Saldo < valor {
        return errors.New("saldo insuficiente")
    }
    c.Saldo -= valor
    return nil
}

func (c *Conta) Creditar(valor float64) {
    c.Saldo += valor
}

type ServicoTransferencia struct{}

func (s *ServicoTransferencia) Transferir(origem *Conta, destino *Conta, valor float64) error {
    if err := origem.Debitar(valor); err != nil {
        return err
    }
    destino.Creditar(valor)
    return nil
}
```

# Exemplo

Abaixo um exemplo de código em Go com todos os itens acima.

```
package main

import (
	"errors"
	"fmt"
	"time"
)

type CPF struct {
	Numero string
}

func NovoCPF(numero string) (*CPF, error) {
	if len(numero) != 11 {
		return nil, errors.New("CPF inválido")
	}
	return &CPF{Numero: numero}, nil
}

type Endereco struct {
	Rua     string
	Cidade  string
	Estado  string
	CEP     string
}

func NovoEndereco(rua, cidade, estado, cep string) Endereco {
	return Endereco{Rua: rua, Cidade: cidade, Estado: estado, CEP: cep}
}

type Cliente struct {
	ID        string
	Nome      string
	CPF       *CPF
	Endereco  Endereco
	CriadoEm  time.Time
	Ativo     bool
}

func NovoCliente(id, nome string, cpf *CPF, endereco Endereco) *Cliente {
	return &Cliente{
		ID:       id,
		Nome:     nome,
		CPF:      cpf,
		Endereco: endereco,
		CriadoEm: time.Now(),
		Ativo:    false,
	}
}

func (c *Cliente) Ativar() {
	c.Ativo = true
}

func (c *Cliente) Desativar() {
	c.Ativo = false
}

type ClienteRepository interface {
	Salvar(cliente *Cliente) error
	BuscarPorID(id string) (*Cliente, error)
}

type ClienteService struct {
	repo ClienteRepository
}

func NovoClienteService(r ClienteRepository) *ClienteService {
	return &ClienteService{repo: r}
}

func (s *ClienteService) CadastrarCliente(id, nome, cpfStr string, endereco Endereco) error {
	cpf, err := NovoCPF(cpfStr)
	if err != nil {
		return err
	}
	cliente := NovoCliente(id, nome, cpf, endereco)
	cliente.Ativar()
	return s.repo.Salvar(cliente)
}

type ClienteRepoMemoria struct {
	dados map[string]*Cliente
}

func NovoClienteRepoMemoria() *ClienteRepoMemoria {
	return &ClienteRepoMemoria{dados: make(map[string]*Cliente)}
}

func (r *ClienteRepoMemoria) Salvar(cliente *Cliente) error {
	r.dados[cliente.ID] = cliente
	return nil
}

func (r *ClienteRepoMemoria) BuscarPorID(id string) (*Cliente, error) {
	cliente, ok := r.dados[id]
	if !ok {
		return nil, errors.New("cliente não encontrado")
	}
	return cliente, nil
}

func main() {
	repo := NovoClienteRepoMemoria()
	service := NovoClienteService(repo)

	endereco := NovoEndereco("Rua A", "São Paulo", "SP", "01000-000")
	err := service.CadastrarCliente("123", "Maria", "12345678901", endereco)
	if err != nil {
		panic(err)
	}

	cliente, err := repo.BuscarPorID("123")
	if err != nil {
		panic(err)
	}

	fmt.Printf("Cliente: %s - Ativo: %v\n", cliente.Nome, cliente.Ativo)
}

```
