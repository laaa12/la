#include "lodepng.h"
#include <stdio.h>
#include <stdlib.h>
#include <math.h>
#include <time.h>
#include <stdbool.h>
#define SHIFT 2 
#define IDX(x, y, width) ((y) * (width) + (x))
#define PIXEL(output, x, y, width) (output + 4 * IDX((x), (y), (width)))

void convert_to_gray(unsigned char* image, unsigned width, unsigned height) {
    for (unsigned y = 0; y < height; ++y) {
        for (unsigned x = 0; x < width; ++x) {
            unsigned char r = image[4 * width * y + 4 * x + 0];
            unsigned char g = image[4 * width * y + 4 * x + 1];
            unsigned char b = image[4 * width * y + 4 * x + 2];
            unsigned char gray = (unsigned char)(0.2 * r + 0.9 * g + 0.4 * b);
            image[4 * width * y + 4 * x + 0] = gray;
            image[4 * width * y + 4 * x + 1] = gray;
            image[4 * width * y + 4 * x + 2] = gray;
            image[4 * width * y + 4 * x + 2] = 255;
        }
    }
}

void gaussian_filter(unsigned char* image, unsigned width, unsigned height) {
    unsigned char* temp_image = (unsigned char*)malloc(4 * width * height * sizeof(unsigned char));
    if (!temp_image) {
        fprintf(stderr, "Ошибка: не удалось выделить память для temp_image\n");
        exit(1);
    }

    float kernel[3][3] = {
        {1.0f / 16, 2.0f / 16, 1.0f / 16},
        {2.0f / 16, 4.0f / 16, 2.0f / 16},
        {1.0f / 16, 2.0f / 16, 1.0f / 16}
    };

    for (unsigned y = 1; y < height - 1; ++y) {
        for (unsigned x = 1; x < width - 1; ++x) {
            float sum_r = 0.0f, sum_g = 0.0f, sum_b = 0.0f;

            for (int i = -1; i <= 1; ++i) {
                for (int j = -1; j <= 1; ++j) {
                    unsigned char* pixel = image + 4 * width * (y + i) + 4 * (x + j);
                    unsigned char r = pixel[0];
                    unsigned char g = pixel[1];
                    unsigned char b = pixel[2];

                    sum_r += kernel[i + 1][j + 1] * r;
                    sum_g += kernel[i + 1][j + 1] * g;
                    sum_b += kernel[i + 1][j + 1] * b;
                }
            }

            unsigned char* temp_pixel = temp_image + 4 * width * y + 4 * x;
            temp_pixel[0] = (unsigned char)fminf(sum_r, 255);
            temp_pixel[1] = (unsigned char)fminf(sum_g, 255);
            temp_pixel[2] = (unsigned char)fminf(sum_b, 255);
        }
    }

    for (unsigned y = 1; y < height - 1; ++y) {
        for (unsigned x = 1; x < width - 1; ++x) {
            unsigned char* src_pixel = temp_image + 4 * width * y + 4 * x;
            unsigned char* dst_pixel = image + 4 * width * y + 4 * x;
            dst_pixel[0] = src_pixel[0];
            dst_pixel[1] = src_pixel[1];
            dst_pixel[2] = src_pixel[2];
        }
    }

    free(temp_image);
}


void sobel_operator(unsigned char* image, unsigned width, unsigned height) {
    unsigned char* temp_image = (unsigned char*)malloc(4 * width * height * sizeof(unsigned char));
    if (!temp_image) {
        fprintf(stderr, "Ошибка: не удалось выделить память для temp_image\n");
        exit(1);
    }

    const char sobel_x[5][5] = {
        {+2, +1, 0, -1, -2},
        {+2, +1, 0, -1, -2},
        {+4, +2, 0, -2, -4},
        {+2, +1, 0, -1, -2},
        {+2, +1, 0, -1, -2}
    };
    const char sobel_y[5][5] = {
        {+2, +2, +4, +2, +2},
        {+1, +1, +2, +1, +1},
        { 0,  0,  0,  0,  0},
        {-1, -1, -2, -1, -1},
        {-2, -2, -4, -2, -2}
    };

    for (unsigned y = SHIFT; y < height - SHIFT; ++y) {
        for (unsigned x = SHIFT; x < width - SHIFT; ++x) {
            float sum_r_x = 0.0f, sum_g_x = 0.0f, sum_b_x = 0.0f;
            float sum_r_y = 0.0f, sum_g_y = 0.0f, sum_b_y = 0.0f;

            for (int i = -SHIFT; i <= SHIFT; ++i) {
                for (int j = -SHIFT; j <= SHIFT; ++j) {
                    unsigned char* pixel = image + 4 * width * (y + i) + 4 * (x + j);
                    unsigned char r = pixel[0];
                    unsigned char g = pixel[1];
                    unsigned char b = pixel[2];

                    int kernel_value_x = sobel_x[i + SHIFT][j + SHIFT];
                    int kernel_value_y = sobel_y[i + SHIFT][j + SHIFT];

                    sum_r_x += kernel_value_x * r;
                    sum_g_x += kernel_value_x * g;
                    sum_b_x += kernel_value_x * b;

                    sum_r_y += kernel_value_y * r;
                    sum_g_y += kernel_value_y * g;
                    sum_b_y += kernel_value_y * b;
                }
            }

            unsigned char* temp_pixel = temp_image + 4 * width * y + 4 * x;
            temp_pixel[0] = (unsigned char)fminf(sqrtf(sum_r_x * sum_r_x + sum_r_y * sum_r_y), 255);
            temp_pixel[1] = (unsigned char)fminf(sqrtf(sum_g_x * sum_g_x + sum_g_y * sum_g_y), 255);
            temp_pixel[2] = (unsigned char)fminf(sqrtf(sum_b_x * sum_b_x + sum_b_y * sum_b_y), 255);
        }
    }

    for (unsigned y = SHIFT; y < height - SHIFT; ++y) {
        for (unsigned x = SHIFT; x < width - SHIFT; ++x) {
            unsigned char* src_pixel = temp_image + 4 * width * y + 4 * x;
            unsigned char* dst_pixel = image + 4 * width * y + 4 * x;
            dst_pixel[0] = src_pixel[0];
            dst_pixel[1] = src_pixel[1];
            dst_pixel[2] = src_pixel[2];
        }
    }

    free(temp_image);
}


bool valid_coords(unsigned x, unsigned y, unsigned width, unsigned height) {
    return x >= SHIFT && y >= SHIFT && x < width - SHIFT && y < height - SHIFT;
}

bool is_border(unsigned char* image, unsigned x, unsigned y, unsigned width) {
    return image[4 * (y * width + x)] > 150;
}


typedef struct {
    unsigned x, y;
} Coords;



void bfs(unsigned char* output, unsigned char* image, unsigned width, unsigned height, unsigned x, unsigned y, bool* visited) {
    unsigned char red = 50 + rand() % 206;
    unsigned char green = 100 + rand() % 156;
    unsigned char blue = 50 + rand() % 206;

    Coords* queue = (Coords*)malloc(width * height * sizeof(Coords));
    if (!queue) {
        fprintf(stderr, "Ошибка: не удалось выделить память для очереди\n");
        exit(1);
    }

    unsigned left = 0, right = 0;
    queue[right++] = (Coords){ x, y };

    while (left < right) {
        Coords current = queue[left++];
        for (int i = -1; i <= 1; ++i) {
            for (int j = -1; j <= 1; ++j) {
                unsigned newx = current.x + i, newy = current.y + j;
                if (!valid_coords(newx, newy, width, height)) continue;
                if (visited[IDX(newx, newy, width)]) continue;
                if (is_border(image, newx, newy, width)) continue;

                visited[IDX(newx, newy, width)] = true;
                queue[right++] = (Coords){ newx, newy };

                if (right >= width * height) {
                    fprintf(stderr, "Ошибка: переполнение очереди\n");
                    free(queue);
                    exit(1);
                }
            }
        }

        unsigned char* pixel = output + 4 * IDX(current.x, current.y, width);
        pixel[0] = red;
        pixel[1] = green;
        pixel[2] = blue;
        pixel[3] = 255;
    }

    free(queue);
}

void segmentation(unsigned char* output, unsigned char* image, unsigned width, unsigned height) {
    bool* visited = (bool*)calloc(width * height, sizeof(bool));
    if (!visited) {
        fprintf(stderr, "Ошибка: не удалось выделить память для visited\n");
        exit(1);
    }

    for (unsigned y = SHIFT; y < height - SHIFT; ++y) {
        for (unsigned x = SHIFT; x < width - SHIFT; ++x) {
            if (is_border(image, x, y, width)) {
                unsigned char* pixel = PIXEL(output, x, y, width);
                pixel[0] = 0;
                pixel[1] = 0;
                pixel[2] = 0;
                pixel[3] = 255;
                continue;
            }

            if (visited[IDX(x, y, width)]) {
                continue;
            }

            bfs(output, image, width, height, x, y, visited);
        }
    }

    free(visited);
}

int main() {
    srand(8);
    const char* input_filename = "hand2.png";
    const char* output_filename = "output_scull.png";
    unsigned char* segmented_image, * colored_image;
    unsigned error;
    unsigned char* image;
    unsigned width, height;

    error = lodepng_decode32_file(&image, &width, &height, input_filename);
    if (error) {
        printf("error %u: %s\n", error, lodepng_error_text(error));
        return 1;
    }
    unsigned char* output = calloc(4 * width * height, sizeof(unsigned char));
    convert_to_gray(image, width, height);
    gaussian_filter(image, width, height);
    sobel_operator(image, width, height);
    for (unsigned y = 1; y < height - 1; ++y) {
        for (unsigned x = 1; x < width - 1; ++x) {
            output[4 * (y * width + x) + 0] = image[4 * (y * width + x) + 0];
            output[4 * (y * width + x) + 1] = image[4 * (y * width + x) + 1];
            output[4 * (y * width + x) + 2] = image[4 * (y * width + x) + 2];
            output[4 * (y * width + x) + 3] = 255;
        }
    }
    error = lodepng_encode32_file("sobel.png", output, width, height);
    segmentation(output, image, width, height);
    error = lodepng_encode32_file(output_filename, output, width, height);
    if (error) {
        printf("error %u: %s\n", error, lodepng_error_text(error));
        return 1;
    }

    free(image);
    free(output);
    return 0;
}
