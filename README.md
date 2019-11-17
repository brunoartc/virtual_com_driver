
Um driver de linux é um programa que facilita a comunicação dos programas com a parte fisica do nosso dispositivo 


Para entendermos como isso funciona primeiro temos que entender o que significam os programas e o kernel

Os programas rodam em uma parte especial da nossa memoria chamada de espaço de usuario, como por exemplo o python ou ate mesmo um bash, esses programas normalmente nao precisam acessar itens de mais baixo nivel como por exxemplo acesso direto a memoria ou portas UBS por exemplo.

Essas tarefas sao executadas pelo kernel space que sabe lidar com o nosso hardware de uma maneira efficiente nos dando uma "API" para isso, são aqui que sao localizados os drivers compilados juntamente ao linux

Os programas localizados no user space tem um momento especifico para pedir coisas para o kernel sobre o hardware como eh o caso de exploradores de arquivos e comunicação serial, caso algum desses programas queira se comunicar com um acessorio que nao tenha seu proprio driver, temos um problema pois esse nao conseguira ser comunicado

Entao temos outra opcao, desenvolver nosso proprio driver, ou um driver quer compila juntamente com o kernel e vem juntamente com ele, ou um simples modulo que é compilado a parte e depois carregado no linux a partir do user space que ensinaremos nesse tutorial




Para fazer um driver do linux temos duas opções, podemos fazer um modulo que seja compilado junto ao kernel ou simplesmente fazer um driver para ser carregado como um modulo vindo do user-space e é isso que vamos fazer desenvolver um driver para fazer uma conn virtual que comunique com seu pc

para isso precisamos de um codigo fonte que sera feito em c

primeiramente precisamo incluir a biblioteca de modulos do linux, para faremos um novo arquivo chamado de tcom.c

```sh
touch tcom.c
```

precisamos adicionar os headers no nosso arquivo tambem
```sh
nano tcom.c
```

dentro do arquivo temos que adicionar as duas linhas de bibliotecas para que possamos usar as funções que conversam com o kernel alem de poder usar seus macros

```C
#include <linux/init.h>
#include <linux/module.h>


```

com as bibliotecas importadas podemos começar o nosso codigo como por exemplo fazer dizer o que nosso driver fará ao ser carregado e descarregado.

https://github.com/torvalds/linux/tree/f1f2f614d535564992f32e720739cb53cf03489f/include/linux/module.h#L72-L74


 essas funções tem um padrão para se seguir em que a função de inicialização retorna um inteiro e o exit nao retorna nada
como mostrado acima


```C
static int init_com(void)
{
    return  0;
}
   
static void finish_com(void)
{
    return;
}

module_init(init_com);
module_exit(finish_com);

```

essas duas funcoes podem ser chamadas pelas macros module_init e module_exit, que vao ser chamados no momento de inicio do modulo e saida do modulo

https://github.com/torvalds/linux/blob/f1f2f614d535564992f32e720739cb53cf03489f/include/linux/module.h#L76-L107

tem tambem muitas outra opções para definir o autor do módulo e ate mesmo a licensa dele, mas nao entraremos em muitos detalhes


Podemos por exemplo fazer um driver que simplesmente mande um Hello World para o kernel quando ele é inicializado para isso semelhante a um programa em C utilizaremos uma função print

precisamos importar mais uma biblioteca para que essa função fique disponivel

```C
#include <linux/kernel.h>
```

com essa funcão importada temos acesso ao printk uma função que printa ao log do kernel


Ate agora temos um programa assim

```C
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/module.h>

static int init_com(void)
{
    printk("\nHello World\n");
    return  0;
}
   
static void finish_com(void)
{
    return;
}

module_init(init_com);
module_exit(finish_com);
```


podemos entao compilar o nosso programa usando uma Makefile contendo


```Makefile
obj-m := tcom.o
```


usamos tambem por padrao o lib modules do nosso sistema que esta localizado em ```/lib/modules/$(uname -r)/build``` 
## se for testar no seu linux deixe /lib/modules/$(uname -r)/build
ou podemos gerar nosso proprio com um kernel do linux 

```bash
make modules_install INSTALL_MOD_PATH=/some/root/folder
```

ou simplesmente dentro da sua pasta do linux

```bash
make modules
```

usando

```bash
make -C /path/para/source/linux M=`pwd` modules

```

para compilar

com isso teremos um arquivo .ko que para todos os fins é o nosso driver, este pode ser carregado e descarregado usando insmod e rmmod respectivamente

como uma simples demosntração podemos abrir um terminal e observar os logs do linux

```bash
tail -f /var/log/kern.log
```

enquanto em outro terminal compilamos o modulo e carregamos ele

```bash
make -C /path/para/source/linux M=`pwd` modules
sudo insmod ./tcom.ko
sudo rmmod ./tcom.ko

```

depois de executar esses comandos no linux voltamos ao nosso terminal que esta acompanhando o log do kernel e vemos que com sucesso obtivemos a mensagem

```log
kernel: [666.1337] Hello World
```

Com isso temos a nossa primeira implementacao de um driver que simplesmesnte sobe uma mensagem para o kernel

Agora precisamos dar alguma funcionalidade para o nosso driver, como por exemplo interfacear com o hardware para simplesmente receber mensagens

Como tudo no linux precisamos abrir um arquivo para fazer a comunicacao com o kernel space e este fazer a comunicacao com o hardware, para isso precisamos importar mais algumas bibliotecas 

```C
#include <linux/config.h>

/* para os codigos de erros que serão utilizados daqui para a frente */
#include <linux/errno.h> 

/* controlar o sistema de arquivos */

#include <linux/fs.h>
#include <linux/proc_fs.h>


/* alocar memoria no kernel space */
#include <linux/slab.h> 

/* medir o tamanho das variaveis usado no kmalloc */
#include <linux/types.h> 


#include <linux/fcntl.h> /* O_ACCMODE */

/* acessar o user space */
#include <asm/uaccess.h> 


```

com esses includes novos podemos trabalhar com algumas funcoes mais avancadas da criacao de drivers e dar mais um passo ao nosso vriver que interpreta mensagens do hardware fisico mas ainda preisamos fazer as funcoes que permitem que nosso driver funcione, no caso para excrever e ler apenas um char, comecaremos declarando qual a regiao de memoria que iremos acessar com o numero de identificação do driver para ele acessar os perifericos que utilizam desse driver

```C

/* o numero da versao do nosso driver */
int memory_major = 60;

/* onde nossa memoria vai ser salva*/

char *memory_buffer;

```

temos tambem que mudar nossa função de inicialização para que quando o driver se inicialize ele aloque um espaço de memoria que será usado, assim como um programa em C, sua liberação tambem deve ser criada, nossa função de inicialização e saida deve ser mais ou menos assim

```C
int memory_init(void) {

  int result;



  /* Registrar o nosso driver para os hardwares certos */

  result = register_chrdev(memory_major, "memory", &memory_fops);


    /* Alocar o espaço de memoria para o programa*/
  memory_buffer = kmalloc(1, GFP_KERNEL); 

  if (!memory_buffer) { 

    result = -ENOMEM;

    memory_exit(); 

    return result;

  } 

  memset(memory_buffer, 0, 1);



  printk("<1>Driver simples de memoria inicializado\n"); 

  return 0;


    

}


void memory_exit(void) {

  /* Liberando o numero de registro no sistema*/

  unregister_chrdev(memory_major, "memory");



  /* Liberando o espaço de memoria */

  if (memory_buffer) {

    kfree(memory_buffer);

  }



  printk("<1>Liberando memorias e descarregando modulo\n");



}

```



temos agora uma parte do nosso programa para iniciar e liberar espaços de memoria que podemos trabalhar, mas nosso driver ainda nao executa nenhuma função 

para isso faremos uma funcao que abre o nosso "arquivo" e uma que deixa nosso arquivo disponivel para outros programas apos a execução da primeira

```C
int memory_open(struct inode *inode, struct file *filp) {



  /* Success */

  return 0;

}
int memory_release(struct inode *inode, struct file *filp) {

 

  /* Success */

  return 0;

}
```


agora tambem precisamos que apos abrir o local de memoria do nosso driver nos possamos ler e escrever nele podemos fazer isso com uma função que copia do buffer para a nossa memoria e vice-versa

```C
ssize_t memory_read(struct file *filp, char *buf, 

                    size_t count, loff_t *f_pos) { 

 

  /* Transfering data to user space */ 

  copy_to_user(buf,memory_buffer,1);



  /* Changing reading position as best suits */ 

  if (*f_pos == 0) { 

    *f_pos+=1; 

    return 1; 

  } else { 

    return 0; 

  }

}
ssize_t memory_write( struct file *filp, char *buf,

                      size_t count, loff_t *f_pos) {



  char *tmp;



  tmp=buf+count-1;

  copy_from_user(memory_buffer,tmp,1);

  return 1;

}

```
