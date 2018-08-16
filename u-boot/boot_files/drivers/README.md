## N. Преобразование строки в беззнаковое целое.

- Функция `simple_strtoul()` конвертирует строку в беззнаковое целое число (unsigned long)
	
	```C
	unsigned long simple_strtoul (const char *string, char **endptr, unsigned int base);
	
	// const char *string - строка для выполнения преобразования
	// char **endptr - cсылка на объект типа char*, значение которой содержит адрес следующего символа в строке string, сразу
	//		    после предыдущего найденного числа. Если этот параметр не используется, он должен быть нулевым указателем.
	
	// basis - основание системы исчисления.
	```

	Функции `simple_strtoul()` работают следующим образом. Во-первых, минуются все специальные символы (пробелы, символы табуляции и т.д.). Затем читается каждый символ, входящий в представление числа. Если какой-то символ не может входить в представление числа, то чтение символов прекращается. Таким символом может быть пробел, знак пунктуации и другой нецифровой символ. Если при этом у исходной строки остается остаток от преобразования, то на него будет указывать параметр ___endptr___. Это означает, что если вызвать функцию `simple_strtoul()` с передачей ей в качестве параметра `100.00 Pliers`, то будет возвращена величина `100L`, а параметр ___endptr___ будет указывать на пробел перед словом "Pliers".

```diff
- Функция `simple_strtoul()` устарела.
+ Сейчас необходимо использовать функцию kstrtoul().
```	
- Функция `kstrtoul()` таким же образом.

	```C
	int kstrtoul(const char *string, unsigned int base, unsigned long *res);
	
	// const char *string - строка для выполнения преобразования. Строка не может начинаться со знака "-"
	// unsigned int base - основание системы счисления. Максимальное поддерживаемое основание - 16. Если base задано как 0, то
	// 			основание определяется автоматически с помощью условной семантики. Если строка начинается с 0x, число
	// 			будет анализироваться как шестнадцатеричное (без учета регистра), если оно начинается с 0, то оно будет
	// 			анализироваться как восьмеричное число. В противном случае число будет анализироваться с десятичным
	// 			основанием.
	
	// unsigned long *res - сюда записывается результат в случае выполнения функции преобразования. 
	```
	
	Функция возвращает 0 при успешном выполнении, -ERANGE при переполнении и -EINVAL при ошибке синтаксического анализа строки.


_______________________________________

## 1. Создание и удаление виртуального файла в /proc

```diff
- Сейчас вместо create_proc_entry используется proc_create !!!
```

- Для того, чтобы создать виртуальный файл в файловой системе `/proc`, используется функция ___proc_create___.
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
 
## 2. Форматированный вывод printf()

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
 
 
## 3. Использование памяти ввода/вывода
 
- Основным механизмом используемым для обмена данными с устройствами, являются связанные с памятью регистры и память устройства. Оба называются памятью ввода/вывода, т.к. разница между регистрами и памятью для программного обеспечения не видна.  

- Области памяти ввода/вывода должны быть выделены до начала использования.

  - Прототип функции для получения области памяти является:
	
	```C
	struct resource *request_mem_region(unsigned long start, unsigned long len, char *name);
	
	char *name		// Имя устройства, за которым будут закреплены порты.
	```

	Эта функция выделяет область памяти ___len___ байт, начиная со ___start___. Если все идет хорошо, то функция возвращает ___не NULL___ указатель.

  - Когда больше не нужна выделенная область памяти, она должна быть освобожденна с помощью:
	
	```C
	void release_mem_region(unsigned long start, unsigned long len);
	```

- Вся выделенная для ввода/вывода память перечислена в `/proc/iomem`. Определение искать в `<linux/ioport.h>`.
____________________________________________

## 4. Использование IOREMAP/IOUNMAP

- После того как память выделена, необходимо сделать ремапинг физического адреса hardware IP с виртуальным адресом в пространстве ядра. Делается это с помощью функции  ___ioremap___.

  - Прототип функции ___ioremap___:
	
	```C
	void *ioremap(unsigned long phys_addr, unsigned long size);
	```
	
  - Виртуальный адрес, полученный с помощью ioremap, ___освобождается___ вызовом ___iounmap___.
	
	```C
	void iounmap(void *addr); 
	```
	
- Функции определены в файле `<asm/io.h>`

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

## 5. Соединение физического устройства из Device Tree с драйвером

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
```diff
- Исправить для моего варианта !!!
```
- Необходимо чтобы операционная система идентифицировала соответствующий драйвер для нашего устройства, которое описано в ___DTS___.
  - Сначала необходимо получить доступ к устройству из ___DTS___. Делается это следующим образом:
    
	```C
	static const struct of_device_id myled_of_match[] = {
		{.compatible = "xlnx,myled-1.0"},
     		{},
	};
	```

	Что происходит в данном случае? Имя `myled_of_match[0].compatible` сравнивается со свойством узла,который описывает наше устройство, в DTS. Если совпадение произошло, то это устройство подсоединятся к ___виртуальной шине___ (___platform bus___).

  - Драйвер запрашивает устройство с таким же именем на шине.
 	```C
	static struct platform_driver myled_driver = {
		.driver = {
        		.name = DRIVER_NAME,
         		.owner = THIS_MODULE,
            		.of_match_table = myled_of_match},
     		.probe = myled_probe,
     		.remove = myled_remove,
     		.shutdown = myled_shutdown
	};
	```
	
	Самая важная здесь строчка `.of_match_table = myled_of_match`. Благодаря ей драйвер понимает какое устройство он ищет на шине.
*** 
- Вообще шины можно разделить на два типа.

  - ___Discover-able___  
Такие шины, как PCI и USB, имеют встроенную способность обнаружения внешнего устройства. Когда устройство подключено к шине, оно получает уникальный идентификатор, который будет использоваться для дальнейшей связи с процессором. Устройство, сидящее на шине PCI/USB, может сообщить системе свое имя и где находятся его ресурсы (регистры и т.д). Таким образом, ядро автоматически понимает какие устройства ему доступны и каой использовать драйвер для найденного устройства.

  - ___Non discover-able___  
Embedded systems обычно не имеют сложных шин, которые могли бы автоматически найти устройство, которое присоединили к такой шине. Например: шины типа i2c или SPI. Таким образом, `Platform devices` сами по себе не поддаются обнаружению. Т.е. устройство не может сказать "Эй! Я присутствую!" программному обеспечению. Что делать?  
Для это была введена ___виртуальная шина___ (___platform bus___). С одной стороны устройства подключаются к такой шине, а с другой - к шине присоединяются драйвера, которые запрашивают устройства с необходимым именем.

- Рассмотрим второй случай на примере.
  - Начнем с объявления своей структуры данных. В ней определим ресурсы, используемые нашим устройством.
  
	```C
	struct test_platform_data {
		char *name;
		int id;
		int bus_id;
	};
	```

  - Сейчас я заполню экземпляр этой структуры соответствующими данными.

	```C
	static struct test_platform_data = {
		.name = "test device data",
		.id = 0,
		.bus_id = -1,
	};
	```

  - Теперь пришло время определить `Platform device`. В ядре Linux объявляется `struct platform_device` для регистрации устройства.

	```C
	ststic struct platform_device drivertest_device = {
		.name = "drivertest",
		.id = 0,
		.dev = {
			.release = drivertest_device_release,
			.platform_data = &psudo_data,
		},
	};
	```
	
	___Важным моментом является название устройства___. В моем случае имя моего устройства "drivertest". С помощью функции, которая показана ниже, зарегистрируем устройство.

	```C
	platform_device_register(&drivertest_device);
	```

  - Структура `struct platfrom_driver` используется для регистрации драйвера для устройства.

	```C
	static struct platform_driver drivertest_driver = {
		.driver = {
			.name = "drivertest",
			.owner = THIS_MODULE,
		},
		.probe = drivertest_probe,
		.remove = drivertest_remove,
	};
	```
	
	Имя драйвера должно совпадать с именем устройства. Linux kernel сравнивает имя драйвера с ранее определенным именем устройства. Если совпадение найдено, то вызывается функция `drivertest_probe`. Эта функция отвечает за инициализацию устройства.
	
  - В приведенном ниже фрагменте показана подпрограмма для функции `drivertest_probe`.

	```C
	static int drivertest_probe(struct platform_device *dev)
	{
		int ret;
		struct test_platform_data *p = (struct test_platform_data *)dev->dev.platform_data;
		printk(KERN_ALERT "Platform data->name: %s\n", p->name);
		printk(KERN_ALERT "Platform data->id: %d\n", p->id);
		printk(KERN_ALERT "Platform data->bus_id: %d\n", p->bus.id);
		printk(KERN_ALERT "Probe device: %s\n", dev->name);
		...
	};

	//https://kerneltweaks.wordpress.com/2014/03/30/
	```
	
	В функции `drivertest_probe` я извлекаю данные о platform_data, которые назначаются устройству во время загрузки. Эта информация содержит всю информацию об устройстве. Результат показан ниже:

	```console
	[118971.277307] Platform data->name: test device data
	[118971.277310] Platform data->id: 0
	[118971.277312] Platform data->bus_id: -1
	[118971.277314] Probe device: drivertest
	```
_____________________________________________

## 6. Автозагрузка модуля

- Автоматическая загрузка модулей ядра это удобная функция, поддерживаемая в Linux.

  - Во время компиляции информация о поддержке устройства генерируется в объекте модуля драйвера. Драйвер начинает идентифицировать устройства, которые он знает. Взгляните на драйвер, который поддерживает карту Xircom CardBus Ethernet (`drivers/net/tulip/xircom_cb.c`) и найдите этот фрагмент:

	```C
	static struct pci_device_id xircom_pci_table[] = {
		{0x115D, 0x0003, PCI_ANY_ID, PCI_ANY_ID,},
  		{0,},
	};

	MODULE_DEVICE_TABLE(pci, xircom_pci_table);
	```
	
	Этот код показывает, что драйвер может поддерживать любую карту, имеющую __PCI vendor ID__ 0x115D и __PCI device ID__ 0x0003. Когда устанавливается модуль драйвера, утилита `depmod` просматривает изображение модуля и расшифровывает идентификаторы, присутствующие в таблице устройств. Затем эта утилита добавляет следующую запись в `/lib/modules/kernel-version/modules.alias` (__v обозначает VendorID, d обозначает DeviceID, sv обозначает subvendorID__):

	```console
	alias pci:v0000115Dd00000003sv*sd*bc*sc*i* xircom_cb
	```

  - Когда вы включаете карту Xircom в гнездо CardBus, ядро генерирует событие ___uevent___, которое идентифицирует недавно добавленное устройство. Вы можете посмотреть созданные события с помощью `udevmonitor`:

	```console
	$ udevmonitor ––env
	...
	MODALIAS=pci:v0000115Dd00000003sv0000115Dsd00001181bc02sc00i00
	...
	```
	
  - Затем `udevd` получает `uevents` через сетевые сокеты. Событие `udevd` вызывает `modprobe` с информацией `MODALIAS`, которую ядро передало `udevd`:

	```console
	modprobe pci:v0000115Dd00000003sv0000115Dsd00001181bc02sc00i00
	```

  - `modprobe` ищет совпадение в `/lib/modules/kernel-version/modules.alias` и переходит к вставленному `xircom_cb`:

	```console
	$ lsmod
	Module      Size   Used by
	xircom_cb   10433  0
	...
	```

- После этого карта готова к использованию.
_____________________________________________

## 7. Регистрация драйвера

- Регистрация дравера происходит с помощью макроса, который приведен ниже.
	
	```C
	#define module_platform_driver(__platform_driver)	module_driver(__platform_driver, platform_driver_register, platform_driver_unregister)
	```

_____________________________________________

## Пример

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

______________________________________________

## 8. Информационные поля

Неплохо было бы дать пользователю возможность понять, что и как делает модуль, и как связаться с автором. Для этого в `module.h` определены макросы `MODULE_DESCRIPTION` и `MODULE_AUTHOR`. Получить информацию об авторе можно командой `modinfo -a` , описание модуля - `modinfo -d`.



______________________________________________

<h1 align="center">Linux Data Structures</h1>

## struct resource

Диапазон выделяемых ресурсов описывается через структуру ___struct resource___, которая объявлена в заголовочном файле `<linux/ioport.h>`:
```C
 struct resource {
 	const char *name;
  	unsigned long start, end;
  	unsigned long flags;
  	struct resource *parent, *sibling, *child;
 };
 ```
Глобальный (корневой) диапазон ресурсов создается во время загрузки. Например, структура ресурсов, описывающая порты ввода/вывода создается следующим образом:
```C
 struct resource ioport_resource = { "PCI IO", 0x0000, IO_SPACE_LIMIT, IORESOURCE_IO };
```
Здесь описан ресурс с именем PCI IO, который покрывает диапазон адресов от нуля до IO_SPACE_LIMIT. Значение данной переменной зависит от используемой платформы и может быть равен 0xFFFF (16-ти битовое адресное пространство, для архитектур x86, IA-64, Alpha, M68k и MIPS), 0xFFFFFFFF (32-х битное пространство, для SPARC, PPC, SH) или 0xFFFFFFFFFFFFFFFF (64-х битное, SPARC64).  

Преимуществом такого управления ресурсами является разделение адресного пространства на поддиапазоны, которые отражают реальную взаимосвязь оборудования. Менеджер ресурсов ___не может выделить пересекающиеся поддиапазоны адресов___, что может предотвратить установку неверно работающего драйвера.

















Код для связывания устройства с драйвером приведен ниже.









