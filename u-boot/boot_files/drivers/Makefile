obj-m := myled.o

all:
	make -C ../../linux-xlnx/ M=$(PWD) modules

clean:
	make -C ../../linux-xlnx M=$(PWD) clean
