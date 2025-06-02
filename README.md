# PR8

## Завдання 8.1
### Чи може write() повертати неповну к-сть байтів після запису у якийсь файловий дискриптор?
### Так, це абсолютно реально з тієї причини, що ресурс не завжди може прийняти всі 100% даних, і через це повертаються лише та к-сть даних, яка змогла записатися.
### Ось наочний приклад:
```
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>
#include <string.h>
#include <errno.h>

int main() {
    int pipefd[2];
    if (pipe(pipefd) == -1) {
        perror("pipe");
        exit(1);
    }

    // Робимо pipe писання неблокуючим
    int flags = fcntl(pipefd[1], F_GETFL);
    fcntl(pipefd[1], F_SETFL, flags | O_NONBLOCK);

    char big_buffer[65536];  // 64 Кб
    memset(big_buffer, 'A', sizeof(big_buffer));

    // Перший write — повністю заповнюємо буфер pipe
    ssize_t count1 = write(pipefd[1], big_buffer, sizeof(big_buffer));
    if (count1 == -1) {
        perror("write first");
        exit(1);
    }
    printf("Перший запис: %ld байт\n", count1);

    // Другий write — намагаємося записати ще 64 Кб, але pipe вже заповнений
    ssize_t count2 = write(pipefd[1], big_buffer, sizeof(big_buffer));
    if (count2 == -1) {
        if (errno == EAGAIN || errno == EWOULDBLOCK) {
            printf("Другий запис: pipe переповнений, нічого не записано\n");
        } else {
            perror("write second");
        }
    } else if (count2 < sizeof(big_buffer)) {
        printf("Другий запис: частковий запис %ld байт із %lu\n", count2, sizeof(big_buffer));
    } else {
        printf("Другий запис: повний запис %ld байт\n", count2);
    }

    close(pipefd[0]);
    close(pipefd[1]);
    return 0;
}
```
### В цьому коді, після переповнення pipe, відповідно, нічого записати далі не можемо
![alt text](image.png)

## Завдання 8.2
### Одразу дам відповідь на питання: після переміщення покажчика на 3 позиції вперед та зчитування 4 байтів, ми отримаємо у записі 4, 5, 6 та 7 комірки нашого масиву
### Наведемо приклад в коді:
```
#include <stdio.h> // ofr i/o
#include <fcntl.h> // for open(), O_CREAT, O_WRONLY...
#include <unistd.h> // for read(), write(), lseek(), close()
#include <stdlib.h> //

int main() {
    const char *filename = "test.bin"; // write filename
    unsigned char data[10] = {4, 5, 2, 2, 3, 3, 7, 9, 1, 5}; // array

    // Creating file and writing to it 10 bytes
    int fd = open(filename, O_CREAT | O_WRONLY | O_TRUNC, 0644);
    // O_CREAT - create if not exist, O_WRONLY - only write, 
    // O_TRUNC - clear if already exist, 0644 - permision type
    if (fd < 0) { // check for opening
        perror("open for write");
        return 1;
    }

    write(fd, data, 10); // write data to file and close
    close(fd);

    fd = open(filename, O_RDONLY); // open for reading
    if (fd < 0) {
        perror("open for read");
        return 1;
    }

    // moving file pointer on 3 positions forward from current pos (SEEK_SET)
    lseek(fd, 3, SEEK_SET); 

    unsigned char buffer[4];
    ssize_t bytesRead = read(fd, buffer, 4); // read 4 chars after moving pointer

    if (bytesRead != 4) { // check for read error
        perror("read");
        close(fd);
        return 1;
    }

    // print buffer
    printf("Bytes read: ");
    for (int i = 0; i < 4; i++) {
        printf("%u ", buffer[i]);
    }
    printf("\n");

    close(fd);
    return 0;
}
```
### І покажемо ті комірки, які ми зчитали з файлу після lseek():
![alt text](image-1.png)

## Завдання 8.3
### Тут для перевірки найгіршого випадку розташування елементів для швидкого сортування, у коді я обрав 4 випадки:<br>1)Вже відсортований <br>2)Зворотньо відсортований <br>3)Випадкове розташування <br>4)Рівні значення 
### Поглянемо на мій код:
```
#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <string.h>

#define ARR_SIZE 10000

int compare_ints(const void *a, const void *b){
    return (*(int *)a / *(int *)b);
    // we show that our void pointers is actualy use int values
}

void fill_sorted(int *arr, int size){
    for(int i = 0; i < size; i++){ arr[i] = i;}
}

void fill_reverce(int *arr, int size){
    for(int i = 0; i < size; i++){ arr[i] = size - i;}
}

void fill_random(int *arr, int size){
    for(int i = 0; i < size; i++){ arr[i] = rand();}
}

void fill_equal(int *arr, int size){
    for(int i = 0; i < size; i++){ arr[i] = 101;}
}

double time_sort(void (*fill_func)(int*, int), int *arr, int size){
    fill_func(arr, size); // fill array
    clock_t start = clock();
    qsort(arr, size, sizeof(int), compare_ints);
    clock_t end = clock();

    return (double)(end - start) / CLOCKS_PER_SEC;
}

int main(int argc, char const *argv[])
{
    srand(time(NULL));
    int *arr = malloc(ARR_SIZE * sizeof(int));

    printf("Start testing on %d elements...\n", ARR_SIZE);

    double t1 = time_sort(fill_sorted, arr, ARR_SIZE);
    printf("Already sorted: %.6f sec\n", t1);

    double t2 = time_sort(fill_reverce, arr, ARR_SIZE);
    printf("Reverce sorted: %.6f sec\n", t2);

    double t3 = time_sort(fill_random, arr, ARR_SIZE);
    printf("Random sorted: %.6f sec\n", t3);

    double t4 = time_sort(fill_equal, arr, ARR_SIZE);
    printf("Equal sorted: %.6f sec\n", t4);

    free(arr);
    return 0;
}
```
### І нарешті результати:
![alt text](image-2.png)

## Завдання 8.4
### В цьому завданні треба трошки розібратися, що у нас виводить fork().
### При створенні дочірнього процесу за допомогою fork(), функція може повертати негативні значення (якщо створення дочірнього процесу було з помилкою), нуль, якщо це новостворений дочірній процес, або більше нуля - це значення pid дочірнього процесу, яке повертає батьківський при створенні.
### Виконаємо початковий код:
```
#include <stdio.h>
#include <unistd.h>

int main() {
    int pid;
    pid = fork();
    printf("%d\n", pid);
}
```
### І отримаємо результат:
![alt text](image-3.png)

### Спочатку йде pid дочірнього процесу від батьківського, а потім вже йде надсилання pid від самого дочірнього процесу
### Після цього, додамо трошки перевірок та вдосконалимо код:
```
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>

int main(int argc, char const *argv[])
{
    pid_t pid;
    pid = fork();

    if(pid < 0){
        perror("fork failed");
        return 1;
    }
    
    printf("%d\n", pid);

    return 0;
}
```
### І отримаємо аналогічний результат:
![alt text](image-4.png)

## Завдання 8.5 (12 варіант)
### Завданням було створити програму, яка симулює часткове зчитування з фалу з паралельнимзаписом з іншого процесу.
### Реалізував я це через fork() і для визначення процесу я просто поставив умову на pid, що кожен pid (батьківський і дочірній) виконують свою функцію:
```
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>
#include <string.h>
#include <sys/types.h>
#include <sys/wait.h>

#define FILENAME "testfile.txt"
#define CHUNK_SIZE 5

int main() {
    int fd;
    pid_t pid;

    // Create or clear the files
    fd = open(FILENAME, O_CREAT | O_TRUNC | O_RDWR, 0644);
    if (fd < 0) {
        perror("open");
        exit(1);
    }

    // Start data in file
    write(fd, "1234567890", 10);
    lseek(fd, 0, SEEK_SET); // Return to start

    pid = fork();

    if (pid < 0) {
        perror("fork");
        exit(1);
    }

    if (pid == 0) {
        // Child process write new data with dellay
        sleep(1); // Give some time to father to read

        fd = open(FILENAME, O_WRONLY | O_APPEND);
        if (fd < 0) {
            perror("child open");
            exit(1);
        }

        const char *extra = "ABCDEF";
        write(fd, extra, strlen(extra));
        close(fd);

        printf("Child: Add ABCDEF to file\n");
        exit(0);
    } else {
        // father process: read file among the parts
        char buffer[CHUNK_SIZE + 1];
        int bytes_read;

        printf("Parent: Start reading file...\n");

        while ((bytes_read = read(fd, buffer, CHUNK_SIZE)) > 0) {
            buffer[bytes_read] = '\0';
            printf("Parent: Readed: \"%s\"\n", buffer);
            sleep(1); // Delay
        }

        close(fd);
        wait(NULL); // Wait for child process to end
        printf("Parent: Reading completed.\n");
    }

    return 0;
}
```

![alt text](image-5.png)