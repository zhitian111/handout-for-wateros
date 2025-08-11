main.o: main.cpp
    g++ -c main.cpp -o main.o
main : main.o
	g++ main.o -o main