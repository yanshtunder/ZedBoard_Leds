	## Создание и удаление виртуального файла в /proc
	
	Для того, чтобы создать виртуальный файл в файловой системе `/proc`, используется функция ___create_proc_entry___.
	Функция возвращает указатель на структуру ___proc_dir_entry___ (или ___NULL___ в случае возникновения ошибки).
	***
	Прототип функции ___create_proc_entry___ для ___создания___ файла в /proc:
	```C
	struct proc_dir_entry *create_proc_entry (const char *name, mode_t mode, struct proc_dir_entry *parent);
	```
	 
	___const char *name___ - имя созаваемого файла
	___mode_t mode___ - режим доступа к файлу
	___struct proc_dir_entry *parent___ - один из подкаталогов в /proc куда разместится создаваемый файл. Параметр parent принимает значение NULL если файл находится непосредственно в каталоге /proc или другое значение, соответствующее каталогу, в который вы хотите поместить файл.

 	***
 	Фрагмент структуры `proc_dir_entry`:
	```C
	struct proc_dir_entry
	{
 		const char *name;           				// имя виртуального файла
    	mode_t mode;                				// режим доступа
    	uid_t uid;              					// уникальный номер пользователя - владельца файла
    	gid_t gid;           						// уникальный номер группы, которой принадлежит файл

    	struct inode_operations *proc_iops; 		// функции-обработчики операций с inode
    	struct proc_dir_entry *parent;      		// Родительский каталог
    	...
    	read_proc_t *read_proc;         			// функция чтения из /proc
    	write_proc_t *write_proc;       			// функция записи в /proc
    	void *data;             					// указатель на локальные данные
    	atomic_t count;             				// счетчик ссылок на файл
    	...
 	};
 	```
 	***
 	Прототип функции ___remove_proc_entry___ для ___удаления___ файла из /proc:
	```C
	void remove_proc_entry(const char *name, struct proc_dir_entry *parent);
	```
	При вызове в эту функцию передается строка, содержащая имя удаляемого файла и его местоположение в файловой системе /proc (родительский каталог)
	____________________________________________
	 
	## Форматированный вывод printf()

	```C
	printf("A = 0x%08lx and B = 0x%08lx", A, B);
	```
	Если A = 1, а B = 2, то код выведет: A = 0x000.000.01 and B = 0x000.000.02 (Точка здесь только для удобства. В реальности она не будет отображаться)
		
	
	___%08lx___
	 |||
	 |||
	 ||+--- ___l___ - совместно со спецификаторами i, d, o, u, x, X означают длинные целые long и unsigned long.
	 |||
	 ||+--- ___l___ - совместно со спецификаторами s, c означают "широкие" многобайтовые строку и символ соответственно.
	 ||
	 ||
	 |+----- Ширина поля
	 |
	 |
	 +------ Если выводимое значение содержит меньше символов, чем указано, то оно будет дополнено пробелами
			 (или нулями если есть флаг "0") до нужной ширины.
	 
	____________________________________________
	 
	 
	## Использование памяти ввода/вывода
	 
	Основным механизмом используемым для обмена данными с устройствами, являются связанные с памятью регистры и память устройства.
	Оба называются памятью ввода/вывода, т.к. разница между регистрами и памятью для программного обеспечения не видна.
	*** 
	Области памяти ввода/вывода должны быть выделены до начала использования.
	
	Прототип функции для получения области памяти является:
	```C
	struct resource *request_mem_region(unsigned long start, unsigned long len, char *name);
	```
	___char *name___ - Имя устройства, за которым будут закреплены порты.
	***
	Эта функция выделяет область памяти ___len___ байт, начиная со ___start___. Если все идет хорошо, то функция возвращает ___не NULL___ указатель.
	Вся выделенная для ввода/вывода память перечислена в /proc/iomem. Определение искать в <linux/ioport.h>
	***

	Когда больше не нужна выделенная область памяти, она должна быть освобожденна с помощью:
	```C
	void release_mem_region(unsigned long start, unsigned long len);
	```
	____________________________________________

	## Использование IOREMAP/IOUNMAP
	
	___ioremap___ является наиболее полезной для ___связи физического адресса hardware IP с виртуальным адресом в пространстве ядра___.
	void *ioremap(unsigned long phys_addr, unsigned long size);
	***
	Виртуальный адрес, полученный с помощью ioremap, ___освобождается___ вызовом ___iounmap___.
	void iounmap(void *addr); 
	***
	Функции определены в файле <asm/io.h>
	
	____________________________________________
	
	
		
		
		
		
	## Пример:
	``` C	
	static int myled_probe(struct platform_device *pdev)
	{

		struct proc_dir_entry *myled_proc_entry;
		int ret = 0;		// Переменная для отображения ошибки/успеха отработки функции
		...
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
		
		err_create_proc_entry:
			iounmap(base_addr);
		
		err_release_region:
			release_mem_region(res->start, remap_size);
		
		return ret;
	}
	```