# wish.c

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/wait.h>
#include <fcntl.h>
#include <ctype.h>

#define MAX_PATHS 64
#define MAX_ARGS 64
#define MAX_PARALLEL 32

char *search_paths[MAX_PATHS];
int num_paths = 1;

void print_error() {
    char error_message[30] = "An error has occurred\n";
    write(STDERR_FILENO, error_message, strlen(error_message));
}

void init_paths() {
    search_paths[0] = strdup("/bin");
}

void free_paths() {
    for (int i = 0; i < num_paths; i++) {
        if (search_paths[i]) {
            free(search_paths[i]);
            search_paths[i] = NULL;
        }
    }
    num_paths = 0;
}

char *find_command(char *cmd) {
    if (cmd == NULL || strlen(cmd) == 0) return NULL;
    
    static char full_path[256];
    for (int i = 0; i < num_paths; i++) {
        snprintf(full_path, sizeof(full_path), "%s/%s", search_paths[i], cmd);
        if (access(full_path, X_OK) == 0) {
            return full_path;
        }
    }
    return NULL;
}

void builtin_path(char **args) {
    free_paths();
    
    int i = 1;
    while (args[i] != NULL && i < MAX_PATHS) {
        search_paths[num_paths] = strdup(args[i]);
        num_paths++;
        i++;
    }
}

void builtin_cd(char **args) {
    if (args[1] == NULL || args[2] != NULL) {
        print_error();
        return;
    }
    
    if (chdir(args[1]) != 0) {
        print_error();
    }
}

int parse_redirection(char **args, char **output_file) {
    int redirect_count = 0;
    int redirect_pos = -1;
    
    for (int i = 0; args[i] != NULL; i++) {
        if (strcmp(args[i], ">") == 0) {
            redirect_count++;
            redirect_pos = i;
        }
    }
    
    if (redirect_count == 0) {
        *output_file = NULL;
        return 0;
    }
    
    if (redirect_count > 1) {
        return -1;
    }
    
    if (redirect_pos == 0 || args[redirect_pos + 1] == NULL) {
        return -1;
    }
    
    if (args[redirect_pos + 2] != NULL) {
        return -1;
    }
    
    *output_file = args[redirect_pos + 1];
    args[redirect_pos] = NULL;
    
    if (args[0] == NULL) {
        return -1;
    }
    
    return 0;
}

char **parse_command(char *cmd) {
    char **args = malloc(MAX_ARGS * sizeof(char*));
    if (args == NULL) return NULL;
    
    int position = 0;
    char *token = strtok(cmd, " \t\r\n");
    
    while (token != NULL && position < MAX_ARGS - 1) {
        args[position++] = token;
        token = strtok(NULL, " \t\r\n");
    }
    args[position] = NULL;
    
    return args;
}

char *trim_whitespace(char *str) {
    while (isspace((unsigned char)*str)) str++;
    
    if (*str == 0) return str;
    
    char *end = str + strlen(str) - 1;
    while (end > str && isspace((unsigned char)*end)) end--;
    
    *(end + 1) = '\0';
    
    return str;
}

void process_line(char *line) {
    line = trim_whitespace(line);
    
    if (strlen(line) == 0) return;
    
    char *commands[MAX_PARALLEL];
    int num_commands = 0;
    
    char *line_copy = strdup(line);
    char *saveptr;
    char *cmd = strtok_r(line_copy, "&", &saveptr);
    
    while (cmd != NULL && num_commands < MAX_PARALLEL) {
        cmd = trim_whitespace(cmd);
        
        if (strlen(cmd) > 0) {
            commands[num_commands++] = strdup(cmd);
        }
        
        cmd = strtok_r(NULL, "&", &saveptr);
    }
    free(line_copy);
    
    if (num_commands == 0) {
        return;
    }
    
    pid_t pids[MAX_PARALLEL];
    int num_pids = 0;
    int has_error = 0;
    
    for (int i = 0; i < num_commands; i++) {
        char **args = parse_command(commands[i]);
        
        if (args == NULL || args[0] == NULL) {
            if (args) free(args);
            free(commands[i]);
            continue;
        }
        
        if (strcmp(args[0], "exit") == 0) {
            if (args[1] != NULL) {
                print_error();
                has_error = 1;
            } else if (num_commands == 1) {
                for (int j = 0; j < num_commands; j++) {
                    free(commands[j]);
                }
                free(args);
                free_paths();
                exit(0);
            } else {
                print_error();
                has_error = 1;
            }
            free(args);
            free(commands[i]);
            continue;
        }
        
        if (strcmp(args[0], "cd") == 0) {
            if (num_commands > 1) {
                print_error();
                has_error = 1;
            } else {
                builtin_cd(args);
            }
            free(args);
            free(commands[i]);
            continue;
        }
        
        if (strcmp(args[0], "path") == 0) {
            if (num_commands > 1) {
                print_error();
                has_error = 1;
            } else {
                builtin_path(args);
            }
            free(args);
            free(commands[i]);
            continue;
        }
        
        char *output_file = NULL;
        if (parse_redirection(args, &output_file) != 0) {
            print_error();
            free(args);
            free(commands[i]);
            has_error = 1;
            continue;
        }
        
        char *cmd_path = find_command(args[0]);
        if (cmd_path == NULL) {
            print_error();
            free(args);
            free(commands[i]);
            has_error = 1;
            continue;
        }
        
        pid_t pid = fork();
        
        if (pid == 0) {
            if (output_file != NULL) {
                int fd = open(output_file, O_WRONLY | O_CREAT | O_TRUNC, 0644);
                if (fd == -1) {
                    print_error();
                    exit(1);
                }
                dup2(fd, STDOUT_FILENO);
                dup2(fd, STDERR_FILENO);
                close(fd);
            }
            
            execv(cmd_path, args);
            print_error();
            exit(1);
        } else if (pid > 0) {
            pids[num_pids++] = pid;
        } else {
            print_error();
            has_error = 1;
        }
        
        free(args);
        free(commands[i]);
    }
    
    for (int i = 0; i < num_pids; i++) {
        waitpid(pids[i], NULL, 0);
    }
}

void interactive_mode() {
    char *line = NULL;
    size_t bufsize = 0;
    ssize_t nread;
    
    while (1) {
        printf("wish> ");
        fflush(stdout);
        
        nread = getline(&line, &bufsize, stdin);
        
        if (nread == -1) {
            free(line);
            break;
        }
        
        if (nread > 0 && line[nread - 1] == '\n') {
            line[nread - 1] = '\0';
        }
        
        process_line(line);
    }
    
    free(line);
}

void batch_mode(char *filename) {
    FILE *file = fopen(filename, "r");
    if (file == NULL) {
        print_error();
        exit(1);
    }
    
    char *line = NULL;
    size_t bufsize = 0;
    ssize_t nread;
    
    while ((nread = getline(&line, &bufsize, file)) != -1) {
        if (nread > 0 && line[nread - 1] == '\n') {
            line[nread - 1] = '\0';
        }
        
        process_line(line);
    }
    
    free(line);
    fclose(file);
}

int main(int argc, char *argv[]) {
    init_paths();
    
    if (argc == 1) {
        interactive_mode();
    } else if (argc == 2) {
        batch_mode(argv[1]);
    } else {
        print_error();
        exit(1);
    }
    
    free_paths();
    return 0;
}