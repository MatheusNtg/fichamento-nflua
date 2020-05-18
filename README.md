# Fichamento NFLua

## O que é o NFLua?

O NFLua é um binding para o NetFilter, que por sua vez é o firewall do linux. O NFLua permite o registro e a execução de código em lua para os hooks das operações do NetFilter.

## Operações Oferecidas pelo NFLua

- `struct nflua_state *nflua_state_create(struct xt_lua_net *xt_lua,
	size_t maxalloc, const char *name)` para a criação de estados.
- `int nflua_state_destroy(struct xt_lua_net *xt_lua, const char *name)` para a delação de estados.
- `void nflua_states_init(struct xt_lua_net *xt_lua)` para a iniciazliação dos estados (inicializar os spinslocks da estrutura que engloba os estados e inicializar a cabeça de cada um dos buckets utilizados para armazenar os estados).
- `void nflua_state_destroy_all(struct xt_lua_net *xt_lua)` destroi todos os estados presente em um namespace.
- `void nflua_states_exit(struct xt_lua_net *xt_lua)` sai (para de utilizar) do nflua, somente faz uma chamada à função `nflua_state_destroy_all`.
- `int nflua_state_list(struct xt_lua_net *xt_lua,nflua_state_cb cb,unsigned short *total)` utilizada para listar os estados presentes em um namespace.
- `struct nflua_state *nflua_state_lookup(struct xt_lua_net *xt_lua,const char *name)` utilizada para verificar a existência de um determinado estado.

Todas as funções citadas acima estão implementadas no arquivo `src/states.c` e oferecem uma API para a manipulação de estados dentro do ambiente do NFLua.


## A `struct nflua_state`

Definida como: 
```c
struct nflua_state {
	struct hlist_node node;
	lua_State *L;
	struct xt_lua_net *xt_lua;
	spinlock_t lock;
	kpi_refcount_t users;
	u32 dseqnum;
	size_t maxalloc;
	size_t curralloc;
	unsigned char name[NFLUA_NAME_MAXSIZE];
};
```

- `node` utilizado para permitir que a `struct nflua_state` seja armazenada em uma hash table.
- `L` estado lua utilizado para execução de código, armazenamento do contexto etc.
- `xt_lua` referência ao namespace à que a `struct nflua_state` se refere.
- `lock` variável utilizada para a sincronização entre os usuários da struct.
- `users` quantos usuários estão referênciando  este estado (revisar).
- `dseqnum` ainda não sei o propósito desta variável.
- `maxalloc` tamanho do arquivo máximo suportado (revisar).
- `curralloc` tamanho do arquivo atual carregado no estado (revisar).
- `name` nome do estado, utilizado como identificador único.

## A `struct xt_lua_net`

Definida como:

```c
struct xt_lua_net {
	struct sock *sock;
	spinlock_t client_lock;
	spinlock_t state_lock;
	spinlock_t rfcnt_lock;
	atomic_t state_count;
	struct hlist_head client_table[XT_LUA_HASH_BUCKETS];
	struct hlist_head state_table[XT_LUA_HASH_BUCKETS];
};
```

Tem por objetivo inserir os dados presentes na mesma dentro da `struct net` que representa um namespace. A `struct xt_lua_net` armazena todas as informaçõe "globais" necessárias para o funcionamento do NFLua.

- `socket` socket de comunicação utilizado pelo netlink para realizar as operações no espaço de usuário.
- `client_lock` (ainda não sei o propósito dessa variável).
- `state_lock` (ainda não sei o propósito dessa variável).
- `rfcnt_lock` (ainda não sei o propósito dessa variável).
- `state_count` contador de estados presente naquele namespace.
- `client_table` tabela hash dos clientes em determinado namespace (clientes são processos que utilizam o netlink para se comunicar com o kernel).
- `state_table` tabela hash dos estados nflua em determinado namespace.

## NFLua e o IPTables

Dentro do diretório `nflua/iptables` temos um readme que explica como o nflua  é usado para o registro de hooks[^1] no netfilter.

### Registrando Hooks do tipo match

Primeiro precisamos garantir que o plugin de match esteja propriamente instalado, para isso precisamos copiar o arquivo `libxt_LUA.so` para o `XTABLES_LIBDIR`, fazemos isso no ubuntu da seguinte forma:

```bash
cp libxt_lua.so /usr/lib/x86_64-linux-gnu/xtables/
```

Agora temos  que registrar o hook da função, fazemos isso utilizando o userspace plugin do iptables oferecido pelo nflua. Utilizamos a opção -m do iptables que tenta dar um match nessa regra com alguma extensão existente,que está instalada no diretório `XTABLES_LIBDIR`, e  no nosso caso é a extensão do nflua. Um exemplo do registro de tal regra é a seguinte:

```bash
iptables -A INPUT -p icmp -m lua --state teste --function just_a_test -j ACCEPT
```

No entanto, o nflua ainda não sabe onde essa função se encontra definida, para isso é necessário fazer a carga de código com um código onde exista uma função com escopo global com o mesmo nome que foi utilizado no comando do iptables. Uma forma de fazê-lo é utilizando o nfluactl com o comando execute que irá executar um código onde você pode passar ou um arquivo que contenha a função definida ou um código que possua a função definida também.

#### Exemplo:
Crie o arquivo `qualquer_nome.lua` em um diretório e escreva a função `just_a_test` nele com um parâmetro nele, que será o pacote recebido para você realizar a manipulação, e faça o que deseja fazer.

```lua
function just_a_test(pacote)
	print("I'm doing what i've to do")
end
```
Além disso, você pode utilizar o lua packet e o lua memory para realizar a manipulação dos pacotes.

### Registrando hooks do tipo target


Os hooks do tipo target, ao contrário dos hooks do tipo match, são hooks que são ativados quando uma determinada regra do iptables tem como alvo a execução de uma função escrita em lua para decidir o destino (no sentido do que vai acontecer com ele), por isso especificamos o target do iptables (-j) como LUA. 

#### Exemplo:
Primeiro precisamos garantir que o plugin de target esteja propriamente instalado, para isso precisamos copiar o arquivo `libxt_LUA.so` para o `XTABLES_LIBDIR`, fazemos isso no ubuntu da seguinte forma:

```bash
cp libxt_LUA.so /usr/lib/x86_64-linux-gnu/xtables/
```
Uma vez que ele se encontra instalado podemos adicionar uma regra nas tabelas do iptables que tenha como target uma função escrita em lua para ser utilizada em um determinado estado. Um exemplo de uma regra dessa forma é a seguinte:

```bash
iptables -A INPUT -p icmp -j LUA --state teste --function target_test
```

Essa regra nos diz o seguinte: Adicione na cadeia `INPUT` uma regra para o protocolo icmp onde o alvo é a extensão em LUA que irá executar a função `target_test` no estado teste. Em outras palavras, toda as vezes que um pacote icmp chegar na máquina ele será processado pela função `target_test` no estado teste que decidirá o seu fim.


Mas assim como no caso do match, nós precisamos realizar a carga do código que conterá essa função no estado desejado, para isso podemos criar um arquivo com uma função de escopo global com o mesmo nome registrado na regra do iptables e implementar a sua lógica. Uma vez feito isso, para realizar a carga do código podemos utilizar o nfluactl informando o path do arquivo que acabamos de escrever e que contem essa função.


[^1]: Funcionalidade provida pelo software que permite a execução de código do usuário (quem está utilizando o software) em determinadas circunstâncias
