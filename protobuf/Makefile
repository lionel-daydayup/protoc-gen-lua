SRC=pb.c

TARGET=pb.so
CFLAGS=`pkg-config --cflags lua5.4` -std=gnu99
LDFLAGS=`pkg-config --libs lua5.4`

all:$(TARGET)

$(TARGET):$(SRC)
	gcc -O3 -shared -fPIC $(SRC) $(CFLAGS) $(LDFLAGS) -o $@
