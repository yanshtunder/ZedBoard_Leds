## Создание и удаление виртуального файла в /proc

```diff
- Сейчас вместо create_proc_entry используется proc_create !!!
```

Для того, чтобы создать виртуальный файл в файловой системе `/proc`, используется функция ___proc_create___.
Функция возвращает указатель на структуру ___proc_dir_entry___ (или ___NULL___ в случае возникновения ошибки).

- Прототип функции ___proc_create___ для ___создания___ файла в `/proc`:
```C
struct proc_dir_entry *proc_create(const char *name, umode_t mode, struct proc_dir_entry *parent, const struct file_operations *proc_fops);


const char *name			   // имя созаваемого файла

umode_t mode				   // режим доступа к файлу
					   // аргумент "0" устанавливает режим доступа 0444

struct proc_dir_entry *parent		   // Один из подкаталогов в /proc куда разместится создаваемый файл.
					   // Параметр parent принимает значение NULL если файл находится 
					   // непосредственно в каталоге /proc или другое значение, соответствующее
					   // каталогу, в который вы хотите поместить файл.
					   
const struct file_operations *proc_fops    // Этот аргумент указывает какие файловые операции будут выполняться с файлом.
```


- Фрагмент структуры `proc_dir_entry`:
```C
struct proc_dir_entry
{
	const char *name;           			// имя виртуального файла
   	mode_t mode;                			// режим доступа
  	uid_t uid;              			// уникальный номер пользователя - владельца файла
   	gid_t gid;           				// уникальный номер группы, которой принадлежит файл

   	struct inode_operations *proc_iops; 		// функции-обработчики операций с inode
   	struct proc_dir_entry *parent;      		// родительский каталог
   	...
   	read_proc_t *read_proc;         		// функция чтения из /proc
   	write_proc_t *write_proc;       		// функция записи в /proc
   	void *data;             			// указатель на локальные данные
   	atomic_t count;             			// счетчик ссылок на файл
   	...
};
```

- Прототип функции ___remove_proc_entry___ для ___удаления___ файла из `/proc`:
```C
void remove_proc_entry(const char *name, struct proc_dir_entry *parent);
```
При вызове в эту функцию передается строка, содержащая имя удаляемого файла и его местоположение в файловой системе `/proc` (родительский каталог)
____________________________________________
 
## Форматированный вывод printf()

```C
int A = 1;
int B = 2;
printf("A = 0x%08lx and B = 0x%08lx", A, B);
```
```console
Result printf() in console:
A = 0x00000001 and B = 0x00000002
```
```C
%08lx  	// l - совместно со спецификаторами i, d, o, u, x, X означают длинные целые long и unsigned long.   
 	// l - совместно со спецификаторами s, c означают "широкие" многобайтовые строку и символ соответственно.    
 	// 8 - ширина поля
	// Если выводимое значение содержит меньше символов, чем указано, то оно будет дополнено пробелами 
	// (или нулями если есть флаг "0") до нужной ширины.  
``` 
____________________________________________
 
 
## Использование памяти ввода/вывода
 
Основным механизмом используемым для обмена данными с устройствами, являются связанные с памятью регистры и память устройства.
Оба называются памятью ввода/вывода, т.к. разница между регистрами и памятью для программного обеспечения не видна.  
- Области памяти ввода/вывода должны быть выделены до начала использования.

- Прототип функции для получения области памяти является:
```C
struct resource *request_mem_region(unsigned long start, unsigned long len, char *name);

char *name		// Имя устройства, за которым будут закреплены порты.
```

Эта функция выделяет область памяти ___len___ байт, начиная со ___start___. Если все идет хорошо, то функция возвращает ___не NULL___ указатель.
Вся выделенная для ввода/вывода память перечислена в `/proc/iomem`. Определение искать в `<linux/ioport.h>`.


- Когда больше не нужна выделенная область памяти, она должна быть освобожденна с помощью:
```C
void release_mem_region(unsigned long start, unsigned long len);
```
____________________________________________

## Использование IOREMAP/IOUNMAP

После того как память выделена, необходимо сделать ремапинг физического адреса hardware IP с виртуальным адресом в пространстве ядра. Делается это с помощью функции  ___ioremap___.
- Прототип функции ___ioremap___:
```C
void *ioremap(unsigned long phys_addr, unsigned long size);
```
- Виртуальный адрес, полученный с помощью ioremap, ___освобождается___ вызовом ___iounmap___.
```C
void iounmap(void *addr); 
```
Функции определены в файле `<asm/io.h>`

____________________________________________

	
	
	
	
	
## Пример:
``` C	
static int myled_probe(struct platform_device *pdev)
{

	struct proc_dir_entry *myled_proc_entry;
	int ret = 0;		// Переменная для отображения ошибки отработки функции
	
	res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
     	if (!res)
	{
        	dev_err(&pdev->dev, "No memory resource\n");
        	return -ENODEV;
     	}
	
	remap_size = res->end - res->start + 1;
	if (!request_mem_region(res->start, remap_size, pdev->name))
	{
		dev_err(&pdev->dev, "Cannot request IO\n");
		return -ENXIO;
	}

	base_addr = ioremap(res->start, remap_size);
	if (base_addr == NULL)
	{
		dev_err(&pdev->dev, "Couldn't ioremap memory at 0x%08lx\n", (unsigned long)res->start);
		ret = -ENOMEM;
		goto err_release_region;
	}
	
	
	myled_proc_entry = proc_create(DRIVER_NAME, 0, NULL, &proc_myled_operations);
	if (myled_proc_entry == NULL)
	{
		dev_err(&pdev->dev, "Couldn't create proc entry\n");
		ret = -ENOMEM;
		goto err_create_proc_entry;
	}
	
	
	printk(KERN_INFO DRIVER_NAME " probed at VA 0x%08lx\n", (unsigned long)base_addr);
	
	return 0;
	
	
	/* Обработка ошибок */
	err_create_proc_entry:
		iounmap(base_addr);
		
	err_release_region:
		release_mem_region(res->start, remap_size);
	
	return ret;
}
```
__________________________________________

## Соединение физического устройства из Device Tree с дравйвером

- Пусть есть такое описание устройства в `Device Tree`:
```console
auart0: serial@8006a000 {
	Позволяет операционной системе идентифицировать соответствующий драйвер устройства. 
    	compatible = "fsl,imx28-aurt", "fsl,imx23-aurt";
    	Адрес и длина области регистров
    	reg = <0x8006a000 0x2000>;
    	Номер прерывания.
    	interrupts = <112>;
    	Механизм DMA и каналы, с именами.
    	dmas = <&dma_apbx 8>, <&dma_apbx 9>;
    	dma-names = "rx", "tx";
    	Описание тактирования.
    	clocks = <&clks 45>;
    	Устройство не используется.
    	status = "disabled";
};
```
- Необходимо чтобы операционная система идентифицировала соответствующий драйвер для нашего устройства. Обычно для этого физическое устройство статически описывают. Код для связывания устройства с драйвером приведен ниже.

```C
#define DRIVER_NAME "myled"
....

/* device match table to match with device node in device tree */
static const struct of_device_id myled_of_match[] = {
	{.compatible = "xlnx,myled-1.0"},
     	{},
};
 
MODULE_DEVICE_TABLE(of, myled_of_match);
 
/* platform driver structure for myled driver */
static struct platform_driver myled_driver = {
	.driver = {
        	.name = DRIVER_NAME,
         	.owner = THIS_MODULE,
            	.of_match_table = myled_of_match},
     	.probe = myled_probe,
     	.remove = myled_remove,
     	.shutdown = myled_shutdown
};

/* Register myled platform driver */
module_platform_driver(myled_driver);
```

```diff
- Исправить для моего варианта !!!
```
