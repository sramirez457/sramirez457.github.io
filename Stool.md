---
layout: default
title: "Stool"
permalink: /software/stool/


/*Header Inclusions*/
#include <iostream>
#include <GL/glew.h>
#include <GL/freeglut.h>

//GLM Math Header Inclusions
#include <glm/glm.hpp>
#include <glm/gtc/matrix_transform.hpp>
#include <glm/gtc/type_ptr.hpp>

//SOIL image loader Inclusion
#include "SOIL2/SOIL2.h"

using namespace std; //Standard namespace

#define WINDOW_TITLE "Enhancement One" //Window title Macro **REVISED TITLE

/*Shader program Macro*/

#ifndef GLSL
#define GLSL(Version, Source) "#version " #Version "\n" #Source
#endif

/*Variable declarations for shader, window size initialization, buffer and array objects */
GLint stoolShaderProgram, lampShaderProgram, WindowWidth = 800, WindowHeight = 600;
GLuint VBO, StoolVAO, LightVAO, texture;

//Subject position and scale
glm::vec3 stoolPosition(0.0f, 0.0f, 0.0f);
glm::vec3 stoolScale(1.5f);    //REVISED: REDUCED TO ZOOM OUT

//stool and light color
glm::vec3 uTexture(0.6f, 0.5f, 0.75f);
glm::vec3 lightColor(0.961f, 0.961f, 0.961f);    //REVISED **COLOR CHANGED TO WHITE SMOKE FROM WHITE

//Light position and scale
glm::vec3 lightPosition(1.5f, 1.5f, 5.0f);
glm::vec3 lightScale(0.3f);

//Camera position
GLfloat cameraSpeed = 0.0001f;   //Movement speed per frame **REVISED to reduce cameraSpeed
GLchar currentKey;    //Will store key pressed

GLfloat lastMouseX = 400, lastMouseY = 300;   //Locks mouse cursor at the center of the screen
GLfloat mouseXOffset, mouseYOffset, yaw = 0.0f, pitch = 0.0f;   //mouse offset, yaw, and pitch variables
GLfloat sensitivity = 0.001f;   //Used for mouse / camera rotation sensitivity **REVISED to reduce sensitivity
bool mouseDetected = true;   //Initially true when mouse movement is detected

//Global vector declarations
glm::vec3 cameraPosition = glm::vec3(0.0f, 0.0f, 0.0f);   //Initial camera position.
glm::vec3 CameraUpY  = glm::vec3(0.0f, 3.0f, 0.0f);  //Temporary y unit vector
glm::vec3 CameraForwardZ = glm::vec3(0.0f, 0.0f, -1.0f);   //Temporary z unit vector
glm::vec3 front;   //Temporary z unit vector for mouse

//Camera rotation
float cameraRotation = glm::radians(-25.0f);

/*Function prototypes*/
void UResizeWindow(int, int);
void URenderGraphics(void);
void UCreateShader(void);
void UCreateBuffers(void);
void UGenerateTexture(void);
void UKeyboard(unsigned char key, int x, int y);
void UKeyReleased(unsigned char key, int x, int y);
void UMouseMove(int x, int y);

/*Stool Vertex Shader Source Code*/
const GLchar * stoolVertexShaderSource = GLSL(330,

        layout (location = 0) in vec3 position; //Vertex data from Vertex Attrib Pointer 0
        layout (location = 1) in vec3 normal; //VAP position 1 for normals
        layout (location = 2) in vec2 textureCoordinate;

        out vec3 FragmentPos; //For outgoing color / pixels to fragment shader
        out vec3 Normal; //For outgoing normals to fragment shader
        out vec2 mobileTextureCoordinate;

        //Global variables for the transform matrices
        uniform mat4 model;
        uniform mat4 view;
        uniform mat4 projection;

void main(){

        gl_Position = projection * view * model * vec4(position, 1.0f); //transforms vertices to clip coordinates

        FragmentPos = vec3(model * vec4(position, 1.0f)); //Gets fragment / pixel position in world space only (exclude view and projection)

        Normal = mat3(transpose(inverse(model))) *  normal; //get normal vectors in world space only and exclude normal translation properties

        mobileTextureCoordinate = vec2(textureCoordinate.x, 1.0f - textureCoordinate.y); //flips the texture horizontal
    }
);


/*Stool Fragment Shader Source Code*/
const GLchar * stoolFragmentShaderSource = GLSL(330,

        in vec3 FragmentPos; //For incoming fragment position
        in vec3 Normal; //For incoming normals
        in vec2 mobileTextureCoordinate;

        out vec4 stoolColor; //For outgoing stool color to the GPU

        //Uniform / Global variables for object color, light color, light position, and camera/view position
        uniform vec3 lightColor;
        uniform vec3 lightPos;
        uniform vec3 viewPosition;

        uniform sampler2D uTexture; //Useful when working with multiple textures

     void main(){
            /*Phong lighting model calculations to generate ambient, diffuse, and specular components*/

            //Calculate Ambient Lighting
            float ambientStrength = 0.5f; //Set ambient or global lighting strength
            vec3 ambient = ambientStrength * lightColor; //Generate ambient light color

            //Calculate Diffuse Lighting
            vec3 norm = normalize(Normal); //Normalize vectors to 1 unit
            vec3 lightDirection = normalize(lightPos - FragmentPos); //Calculate distance (light direction) between light source and fragments/pixels on
            float impact = max(dot(norm, lightDirection), 0.0); //Calculate diffuse impact by generating dot product of normal and light
            vec3 diffuse = impact * lightColor; //Generate diffuse light color

            //Calculate Specular lighting
            float specularIntensity = 0.2f; //Set specular light strength
            float highlightSize = 32.0f; //Set specular highlight size
            vec3 viewDir = normalize(viewPosition - FragmentPos); //Calculate view direction
            vec3 reflectDir = reflect(-lightDirection, norm); //Calculate reflection vector
            //Calculate specular component
            float specularComponent = pow(max(dot(viewDir, reflectDir), 0.0), highlightSize);
            vec3 specular = specularIntensity * specularComponent * lightColor;

            //Calculate phong result
            vec3 objectColor = texture(uTexture, mobileTextureCoordinate).xyz;
            vec3 phong = (ambient + diffuse + specular) * objectColor;
            stoolColor = vec4(phong, 1.0f);   //Send lighting results to GPU
        }

);


/*Lamp Shader Source Code*/
const GLchar * lampVertexShaderSource = GLSL(330,

        layout (location = 0) in vec3 position; //VAP position 0 for vertex position data

        //Uniform / Global variables for the transform matrices
        uniform mat4 model;
        uniform mat4 view;
        uniform mat4 projection;

        void main()
        {
            gl_Position = projection * view *model * vec4(position, 1.0f); //Transforms vertices into clip coordinates
        }

);

/*Fragment Shader Source Code*/
const GLchar * lampFragmentShaderSource = GLSL(330,

        out vec4 color; //For outgoing lamp color (smaller pyramid) to the GPU

        void main()
        {
            color = vec4(0.678f, 0.847f, 0.902f, 1.0f); //Set color lamp **REVISED TO MATCH NEW BACKGROUND COLOR
        }

);


/*Main Program*/
int main(int argc, char* argv[])
{
    glutInit(&argc, argv);

    glutInitDisplayMode(GLUT_DEPTH | GLUT_DOUBLE | GLUT_RGBA);

    glutInitWindowSize(WindowWidth, WindowHeight);

    glutCreateWindow(WINDOW_TITLE);

    glutReshapeFunc(UResizeWindow);

    glewExperimental = GL_TRUE;

            if (glewInit() != GLEW_OK)
            {
                std::cout<< "Failed to initialize GLEW" << std::endl;
                return -1;
            }

    UCreateShader();

    UCreateBuffers();

    UGenerateTexture();

    glClearColor(0.678f, 0.847f, 0.902f, 1.0f); //Set background color  **REVISED TO A LIGHT BLUE

    glutDisplayFunc(URenderGraphics);

	glutKeyboardFunc(UKeyboard);   //Detects key press

	glutKeyboardUpFunc(UKeyReleased); //Detects key release

	glutPassiveMotionFunc(UMouseMove);    //Detects mouse movement

    glutMainLoop();

    //Destroys Buffer objects once used
    glDeleteVertexArrays(1, &StoolVAO);
    glDeleteVertexArrays(1, &LightVAO);
    glDeleteBuffers(1, &VBO);

    return 0;
}


/*Resizes the window*/
void UResizeWindow(int w, int h)
{
    WindowWidth = w;
    WindowHeight = h;
    glViewport(0, 0, WindowWidth, WindowHeight);
}


/*Renders graphics*/
void URenderGraphics(void)
{
    glEnable(GL_DEPTH_TEST); //Enable z-depth

    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT); //Clears the screen

    GLint modelLoc, viewLoc, projLoc, uTextureLoc, lightColorLoc, lightPositionLoc, viewPositionLoc;

    uTextureLoc = glGetUniformLocation(stoolShaderProgram, "uTexture");
    glUniform1i(uTextureLoc, 0); //texture unit 0

    CameraForwardZ = front;   //Replaces camera forward vector with Radians normalized as a unit vector

    glm::mat4 model;
    glm::mat4 view;
    glm::mat4 projection;

    /*********Use the stool Shader to activate the pyramid Vertex Array Object for rendering and transforming*********/
    glUseProgram(stoolShaderProgram);
    glBindVertexArray(StoolVAO);

    //Transform the stool
    model = glm::translate(model, stoolPosition);
    model = glm::scale(model, stoolScale);

    //Transform the camera
    //view = glm::translate(view, cameraPosition);
    //view = glm::rotate(view, cameraRotation, glm::vec3(0.0f, 1.5f, 0.0f));
    view = glm::lookAt(CameraForwardZ, cameraPosition, CameraUpY);

    //Set the camera projection to perspective
    projection = glm::perspective(45.0f,(GLfloat)WindowWidth / (GLfloat)WindowHeight, 0.1f, 100.0f);

    /*Projection Change*/
    if(currentKey == 'o')
    	projection = glm::ortho(-5.0f, 5.0f, -5.0f, 5.0f, 0.1f, 100.0f);

    //Reference matrix uniforms from the stool Shader program
    modelLoc = glGetUniformLocation(stoolShaderProgram, "model");
    viewLoc = glGetUniformLocation(stoolShaderProgram, "view");
    projLoc = glGetUniformLocation(stoolShaderProgram, "projection");

    //Pass matrix data to the stool Shader program's matrix uniforms
    glUniformMatrix4fv(modelLoc, 1, GL_FALSE, glm::value_ptr(model));
    glUniformMatrix4fv(viewLoc, 1, GL_FALSE, glm::value_ptr(view));
    glUniformMatrix4fv(projLoc, 1, GL_FALSE, glm::value_ptr(projection));

    //Reference matrix uniforms from the stool Shader program for the pyramid color, light color, light position, and camera position
    uTextureLoc = glGetUniformLocation(stoolShaderProgram, "texture");
    lightColorLoc = glGetUniformLocation(stoolShaderProgram, "lightColor");
    lightPositionLoc = glGetUniformLocation(stoolShaderProgram, "lightPos");
    viewPositionLoc = glGetUniformLocation(stoolShaderProgram, "viewPosition");

    //Pass color, light, and camera data to the stool Shader programs corresponding uniforms
    glUniform3f(uTextureLoc, uTexture.r, uTexture.g, uTexture.b);
    glUniform3f(lightColorLoc, lightColor.r, lightColor.g, lightColor.b);
    glUniform3f(lightPositionLoc, lightPosition.x, lightPosition.y, lightPosition.z);
    glUniform3f(viewPositionLoc, cameraPosition.x, cameraPosition.y, cameraPosition.z);

    glActiveTexture(GL_TEXTURE0);
    glBindTexture(GL_TEXTURE_2D, texture);

    glDrawArrays(GL_TRIANGLES, 0, 400); //Draw the primitives / stool

    glBindVertexArray(0); //Deactivate the Pyramid Vertex Array Object

    /***************Use the Lamp Shader and activate the Lamp Vertex Array Object for rendering and transforming ************/
    glUseProgram(lampShaderProgram);
    glBindVertexArray(LightVAO);

    //Transform the smaller stool used as a visual cue for the light source
    model = glm::translate(model, lightPosition);
    model = glm::scale(model, lightScale);

    //Reference matrix uniforms from the Lamp Shader program
    modelLoc = glGetUniformLocation(lampShaderProgram, "model");
    viewLoc = glGetUniformLocation(lampShaderProgram, "view");
    projLoc = glGetUniformLocation(lampShaderProgram, "projection");

    //Pass matrix uniforms from the Lamp Shader Program
    glUniformMatrix4fv(modelLoc, 1, GL_FALSE, glm::value_ptr(model));
    glUniformMatrix4fv(viewLoc, 1, GL_FALSE, glm::value_ptr(view));
    glUniformMatrix4fv(projLoc, 1, GL_FALSE, glm::value_ptr(projection));

    glBindTexture(GL_TEXTURE_2D, texture);

    //Draws the triangles
    glDrawArrays(GL_TRIANGLES, 0, 500);

    glBindVertexArray(0); //Deactivate the Lamp Vertex Array Object

    glutPostRedisplay();

    glutSwapBuffers(); //Flips the back buffer with the front buffer every frame. Similar to GL Flush

}

/*Create the Shader program*/
void UCreateShader()
{
    //Stool Vertex shader
    GLint stoolVertexShader = glCreateShader(GL_VERTEX_SHADER); //Creates the Vertex shader
    glShaderSource(stoolVertexShader, 1, &stoolVertexShaderSource, NULL); //Attaches the Vertex shader to the source code
    glCompileShader(stoolVertexShader); //Compiles the Vertex shader

    //Stool Fragment Shader
    GLint stoolFragmentShader = glCreateShader(GL_FRAGMENT_SHADER); //Creates the Fragment Shader
    glShaderSource(stoolFragmentShader, 1, &stoolFragmentShaderSource, NULL); //Attaches the Fragment shader to the source code
    glCompileShader(stoolFragmentShader); //Compiles the Fragment Shader

    //Stool Shader program
    stoolShaderProgram = glCreateProgram(); //Creates the Shader program and returns an id
    glAttachShader(stoolShaderProgram, stoolVertexShader); //Attaches Vertex shader to the Shader program
    glAttachShader(stoolShaderProgram, stoolFragmentShader); //Attaches Fragment shader to the Shader program
    glLinkProgram(stoolShaderProgram); //Link Vertex and Fragment shaders to the Shader program

    //Delete the Vertex and Fragment shaders once linked
    glDeleteShader(stoolVertexShader);
    glDeleteShader(stoolFragmentShader);

    //Lamp Vertex shader
    GLint lampVertexShader = glCreateShader(GL_VERTEX_SHADER); //Creates the Vertex shader
    glShaderSource(lampVertexShader, 1, &lampVertexShaderSource, NULL); //Attaches the Vertex shader to the source code
    glCompileShader(lampVertexShader); //Compiles the Vertex shader

    //Lamp Fragment shader
    GLint lampFragmentShader = glCreateShader(GL_FRAGMENT_SHADER); //Creates the Fragment shader
    glShaderSource(lampFragmentShader, 1, &lampFragmentShaderSource, NULL); //Attaches the Fragment shader to the source code
    glCompileShader(lampFragmentShader); //Compiles the Fragment shader

    //Lamp Shader Program
    lampShaderProgram = glCreateProgram(); //Creates the Shader program and returns an id
    glAttachShader(lampShaderProgram, lampVertexShader); //Attach Vertex shader to the Shader program
    glAttachShader(lampShaderProgram, lampFragmentShader); //Attach Fragment shader to the Shader program
    glLinkProgram(lampShaderProgram); //Link Vertex and Fragment shaders to the Shader program

    //Delete the lamp shaders once linked
    glDeleteShader(lampVertexShader);
    glDeleteShader(lampFragmentShader);

}

/*Creates the Buffer and Array Objects*/
void UCreateBuffers()
{
	GLfloat vertices[] = {
								//Positions				//Normals           //Texture Coordinates

								//Top of Stool - Top
								-1.0f,  2.0f, -1.0f,     0.0f, 1.0f, 0.0f,	  1.0f, 0.0f,	//2
								 1.0f,  2.0f, -1.0f,     0.0f, 1.0f, 0.0f,    1.0f, 1.0f,	//1
								 1.0f,  2.0f,  1.0f,     0.0f, 1.0f, 0.0f,    0.0f, 1.0f,	//0
								-1.0f,  2.0f, -1.0f,     0.0f, 1.0f, 0.0f,	  0.0f, 1.0f,   //2
								-1.0f,  2.0f,  1.0f,     0.0f, 1.0f, 0.0f,    0.0f, 0.0f,   //3
								 1.0f,  2.0f,  1.0f,     0.0f, 1.0f, 0.0f,	  1.0f, 0.0f,   //0

								 //Top of Stool - Bottom
								-1.0f,  1.7f,  1.0f,     0.0f, -1.0f, 0.0f,   0.0f, 0.0f,	//7
								 1.0f,  1.7f,  1.0f,     0.0f, -1.0f, 0.0f,   1.0f, 0.0f,	//4
								 1.0f,  1.7f, -1.0f,     0.0f, -1.0f, 0.0f,   1.0f, 1.0f,	//5
								 1.0f,  1.7f, -1.0f,     0.0f, -1.0f, 0.0f,   0.0f, 0.0f,   //5
								-1.0f,  1.7f, -1.0f,     0.0f, -1.0f, 0.0f,   1.0f, 0.0f,	//6
								-1.0f,  1.7f,  1.0f,     0.0f, -1.0f, 0.0f,   1.0f, 1.0f,   //7

								 //Top of Stool - Front
								-1.0f,  1.7f,  1.0f,     0.0f, 0.0f, 1.0f,	  0.0f, 0.0f,   //7
								 1.0f,  1.7f,  1.0f,     0.0f, 0.0f, 1.0f,	  1.0f, 0.0f,	//4
								 1.0f,  2.0f,  1.0f,     0.0f, 0.0f, 1.0f,	  1.0f, 1.0f,	//0
								 1.0f,  2.0f,  1.0f,     0.0f, 0.0f, 1.0f,	  1.0f, 1.0f,	//0
								-1.0f,  2.0f,  1.0f,     0.0f, 0.0f, 1.0f,	  0.0f, 1.0f,	//3
								-1.0f,  1.7f,  1.0f,     0.0f, 0.0f, 1.0f,	  0.0f, 0.0f,	//7

								//Top of Stool - Right
								 1.0f,  2.0f,  1.0f,     1.0f, 0.0f, 0.0f,	  1.0f, 0.0f,
								 1.0f,  2.0f, -1.0f,     1.0f, 0.0f, 0.0f,	  0.0f, 0.0f,
								 1.0f,  1.7f, -1.0f,     1.0f, 0.0f, 0.0f,	  0.0f, 1.0f,
								 1.0f,  1.7f, -1.0f,     1.0f, 0.0f, 0.0f,	  1.0f, 0.0f,
								 1.0f,  1.7f,  1.0f,     1.0f, 0.0f, 0.0f,	  0.0f, 0.0f,
								 1.0f,  2.0f,  1.0f,     1.0f, 0.0f, 0.0f,	  0.0f, 1.0f,

								 //Top of Stool - Back
								 1.0f,  2.0f, -1.0f,     0.0f, 0.0f, -1.0f,	  1.0f, 0.0f,
								-1.0f,  2.0f, -1.0f,     0.0f, 0.0f, -1.0f,	  0.0f, 0.0f,
								-1.0f,  1.7f, -1.0f,     0.0f, 0.0f, -1.0f,	  0.0f, 1.0f,
								-1.0f,  1.7f, -1.0f,     0.0f, 0.0f, -1.0f,	  1.0f, 0.0f,
								 1.0f,  1.7f, -1.0f,     0.0f, 0.0f, -1.0f,	  0.0f, 0.0f,
								 1.0f,  2.0f, -1.0f,     0.0f, 0.0f, -1.0f,	  0.0f, 1.0f,

								 //Top of Stool - Left
								-1.0f,  2.0f, -1.0f,     -1.0f, 0.0f, 0.0f,   1.0f, 0.0f,
								-1.0f,  2.0f,  1.0f,     -1.0f, 0.0f, 0.0f,	  0.0f, 0.0f,
								-1.0f,  1.7f,  1.0f,     -1.0f, 0.0f, 0.0f,	  0.0f, 1.0f,
								-1.0f,  1.7f,  1.0f,     -1.0f, 0.0f, 0.0f,	  1.0f, 0.0f,
								-1.0f,  1.7f, -1.0f,     -1.0f, 0.0f, 0.0f,	  0.0f, 0.0f,
								-1.0f,  2.0f, -1.0f,     -1.0f, 0.0f, 0.0f,	  0.0f, 1.0f,

								//Back Right Leg - Top
								 1.0f,  1.7f, -1.0f,     0.0f, 1.0f, 0.0f,	  1.0f, 1.0f,   //5
								 1.0f,  1.7f, -0.7f,     0.0f, 1.0f, 0.0f,	  1.0f, 0.0f,   //8
								 0.7f,  1.7f, -0.7f,     0.0f, 1.0f, 0.0f,	  0.0f, 0.0f,   //9
								 0.7f,  1.7f, -0.7f,     0.0f, 1.0f, 0.0f,	  0.0f, 0.0f,
								 0.7f,  1.7f, -1.0f,     0.0f, 1.0f, 0.0f,	  0.0f, 1.0f,   //10
								 1.0f,  1.7f, -1.0f,     0.0f, 1.0f, 0.0f,	  1.0f, 1.0f,

								 //Back Right Leg - Bottom
								 1.0f, -2.0f, -1.0f,     0.0f, -1.0f, 0.0f,   1.0f, 1.0f,  //20
								 1.0f, -2.0f, -0.7f,     0.0f, -1.0f, 0.0f,   1.0f, 0.0f,  //21
								 0.7f, -2.0f, -0.7f,     0.0f, -1.0f, 0.0f,   0.0f, 0.0f,  //22
								 0.7f, -2.0f, -0.7f,     0.0f, -1.0f, 0.0f,   0.0f, 0.0f,
								 0.7f, -2.0f, -1.0f,     0.0f, -1.0f, 0.0f,   0.0f, 1.0f,  //23
								 1.0f, -2.0f, -1.0f,     0.0f, -1.0f, 0.0f,   1.0f, 1.0f,

								 //Back Right Leg - Front
								 0.7f,  1.7f, -0.7f,     0.0f, 0.0f, 1.0f,	  0.0f, 1.0f,
								 1.0f,  1.7f, -0.7f,     0.0f, 0.0f, 1.0f,	  0.3f, 1.0f,
								 1.0f, -2.0f, -0.7f,     0.0f, 0.0f, 1.0f,	  0.3f, 0.0f,
								 1.0f, -2.0f, -0.7f,     0.0f, 0.0f, 1.0f,	  0.3f, 0.0f,
								 0.7f, -2.0f, -0.7f,     0.0f, 0.0f, 1.0f,	  0.0f, 0.0f,
								 0.7f,  1.7f, -0.7f,     0.0f, 0.0f, 1.0f,	  0.0f, 1.0f,

								 //Back Right Leg - Right
								 1.0f,  1.7f, -0.7f,     1.0f, 0.0f, 0.0f,	  0.0f, 1.0f,
								 1.0f,  1.7f, -1.0f,     1.0f, 0.0f, 0.0f,	  0.3f, 1.0f,
								 1.0f, -2.0f, -1.0f,     1.0f, 0.0f, 0.0f,	  0.3f, 0.0f,
								 1.0f, -2.0f, -1.0f,     1.0f, 0.0f, 0.0f,	  0.3f, 0.0f,
								 1.0f, -2.0f, -0.7f,     1.0f, 0.0f, 0.0f,	  0.0f, 0.0f,
								 1.0f,  1.7f, -0.7f,     1.0f, 0.0f, 0.0f,	  0.0f, 1.0f,

								 //Back Right Leg - Back
								 1.0f,  1.7f, -1.0f,     0.0f, 0.0f, -1.0f,	  0.0f, 1.0f,
								 0.7f,  1.7f, -1.0f,     0.0f, 0.0f, -1.0f,	  0.3f, 1.0f,
								 0.7f, -2.0f, -1.0f,     0.0f, 0.0f, -1.0f,	  0.3f, 0.0f,
								 0.7f, -2.0f, -1.0f,     0.0f, 0.0f, -1.0f,	  0.3f, 0.0f,
								 1.0f, -2.0f, -1.0f,     0.0f, 0.0f, -1.0f,	  0.0f, 0.0f,
								 1.0f,  1.7f, -1.0f,     0.0f, 0.0f, -1.0f,	  0.0f, 1.0f,

								 //Back Right Leg - Left
								 0.7f,  1.7f, -1.0f,     -1.0f, 0.0f, 0.0f,   0.0f, 1.0f,
								 0.7f,  1.7f, -0.7f,     -1.0f, 0.0f, 0.0f,   0.3f, 1.0f,
								 0.7f, -2.0f, -0.7f,     -1.0f, 0.0f, 0.0f,   0.3f, 0.0f,
								 0.7f, -2.0f, -0.7f,     -1.0f, 0.0f, 0.0f,   0.3f, 0.0f,
								 0.7f, -2.0f, -1.0f,     -1.0f, 0.0f, 0.0f,   0.0f, 0.0f,
								 0.7f,  1.7f, -1.0f,     -1.0f, 0.0f, 0.0f,   0.0f, 1.0f,

								 //Front Right Leg - Top
								 1.0f,  1.7f,  1.0f,     0.0f, 1.0f, 0.0f,	  1.0f, 0.0f,   //4
								 1.0f,  1.7f,  0.7f,     0.0f, 1.0f, 0.0f,	  1.0f, 1.0f,   //19
								 0.7f,  1.7f,  0.7f,     0.0f, 1.0f, 0.0f,	  0.0f, 1.0f,   //18
								 0.7f,  1.7f,  0.7f,     0.0f, 1.0f, 0.0f,	  0.0f, 1.0f,
								 0.7f,  1.7f,  1.0f,     0.0f, 1.0f, 0.0f,	  0.0f, 0.0f,   //17
								 1.0f,  1.7f,  1.0f,     0.0f, 1.0f, 0.0f,	  1.0f, 0.0f,

								 //Front Right Leg - Bottom
								 1.0f, -2.0f,  0.7f,     0.0f, -1.0f, 0.0f,   1.0f, 1.0f,  //32
								 1.0f, -2.0f,  1.0f,     0.0f, -1.0f, 0.0f,   1.0f, 0.0f,  //33
								 0.7f, -2.0f,  1.0f,     0.0f, -1.0f, 0.0f,   0.0f, 0.0f,  //34
								 0.7f, -2.0f,  1.0f,     0.0f, -1.0f, 0.0f,   0.0f, 0.0f,
								 0.7f, -2.0f,  0.7f,     0.0f, -1.0f, 0.0f,   0.0f, 1.0f,  //35
								 1.0f, -2.0f,  0.7f,     0.0f, -1.0f, 0.0f,   1.0f, 1.0f,

								 //Front Right Leg - Front
								 0.7f,  1.7f,  1.0f,     0.0f, 0.0f, 1.0f,	  0.0f, 1.0f,
								 1.0f,  1.7f,  1.0f,     0.0f, 0.0f, 1.0f,	  0.3f, 1.0f,
								 1.0f, -2.0f,  1.0f,     0.0f, 0.0f, 1.0f,	  0.3f, 0.0f,
								 1.0f, -2.0f,  1.0f,     0.0f, 0.0f, 1.0f,	  0.3f, 0.0f,
								 0.7f, -2.0f,  1.0f,     0.0f, 0.0f, 1.0f,	  0.0f, 0.0f,
								 0.7f,  1.7f,  1.0f,     0.0f, 0.0f, 1.0f,	  0.0f, 1.0f,

								 //Front Right Leg - Right
								 1.0f,  1.7f,  1.0f,     1.0f, 0.0f, 0.0f,    0.0f, 1.0f,
								 1.0f,  1.7f,  0.7f,     1.0f, 0.0f, 0.0f,	  0.3f, 1.0f,
								 1.0f, -2.0f,  0.7f,     1.0f, 0.0f, 0.0f,	  0.3f, 0.0f,
								 1.0f, -2.0f,  0.7f,     1.0f, 0.0f, 0.0f,	  0.3f, 0.0f,
								 1.0f, -2.0f,  1.0f,     1.0f, 0.0f, 0.0f,	  0.0f, 0.0f,
								 1.0f,  1.7f,  1.0f,     1.0f, 0.0f, 0.0f,	  0.0f, 1.0f,

								 //Front Right Leg - Back
								 1.0f,  1.7f,  0.7f,     0.0f, 0.0f, -1.0f,	  0.0f, 1.0f,
								 0.7f,  1.7f,  0.7f,     0.0f, 0.0f, -1.0f,	  0.3f, 1.0f,
								 0.7f, -2.0f,  0.7f,     0.0f, 0.0f, -1.0f,	  0.3f, 0.0f,
								 0.7f, -2.0f,  0.7f,     0.0f, 0.0f, -1.0f,	  0.3f, 0.0f,
								 1.0f, -2.0f,  0.7f,     0.0f, 0.0f, -1.0f,	  0.0f, 0.0f,
								 1.0f,  1.7f,  0.7f,     0.0f, 0.0f, -1.0f,	  0.0f, 1.0f,

								 //Front Right Leg - Left
								 0.7f,  1.7f,  0.7f,     -1.0f, 0.0f, 0.0f,	  0.0f, 1.0f,
								 0.7f,  1.7f,  1.0f,     -1.0f, 0.0f, 0.0f,	  0.3f, 1.0f,
								 0.7f, -2.0f,  1.0f,     -1.0f, 0.0f, 0.0f,	  0.3f, 0.0f,
								 0.7f, -2.0f,  1.0f,     -1.0f, 0.0f, 0.0f,	  0.3f, 0.0f,
								 0.7f, -2.0f,  0.7f,     -1.0f, 0.0f, 0.0f,	  0.0f, 0.0f,
								 0.7f,  1.7f,  0.7f,     -1.0f, 0.0f, 0.0f,	  0.0f, 1.0f,

								 //Front Left Leg - Top
								-0.7f,  1.7f,  1.0f,     0.0f, 1.0f, 0.0f,    1.0f, 0.0f,   //16
								-0.7f,  1.7f,  0.7f,     0.0f, 1.0f, 0.0f,    1.0f, 1.0f,   //15
								-1.0f,  1.7f,  0.7f,     0.0f, 1.0f, 0.0f,    0.0f, 1.0f,   //14
								-1.0f,  1.7f,  0.7f,     0.0f, 1.0f, 0.0f,	  0.0f, 1.0f,
								-1.0f,  1.7f,  1.0f,     0.0f, 1.0f, 0.0f,    0.0f, 0.0f,   //7
								-0.7f,  1.7f,  1.0f,     0.0f, 1.0f, 0.0f,	  1.0f, 0.0f,

								//Front Left Leg - Bottom
								-0.7f, -2.0f,  1.0f,     0.0f, -1.0f, 0.0f,   1.0f, 0.0f,   //29
								-0.7f, -2.0f,  0.7f,     0.0f, -1.0f, 0.0f,   1.0f, 1.0f,   //28
								-1.0f, -2.0f,  0.7f,     0.0f, -1.0f, 0.0f,   0.0f, 1.0f,   //31
								-1.0f, -2.0f,  0.7f,     0.0f, -1.0f, 0.0f,	  0.0f, 1.0f,
								-1.0f, -2.0f,  1.0f,     0.0f, -1.0f, 0.0f,   0.0f, 0.0f,   //30
								-0.7f, -2.0f,  1.0f,     0.0f, -1.0f, 0.0f,	  1.0f, 0.0f,

								//Front Left Leg - Front
								-1.0f,  1.7f,  1.0f,     0.0f, 0.0f, 1.0f,	  0.0f, 1.0f,
								-0.7f,  1.7f,  1.0f,     0.0f, 0.0f, 1.0f,	  0.3f, 1.0f,
								-0.7f, -2.0f,  1.0f,     0.0f, 0.0f, 1.0f,	  0.3f, 0.0f,
								-0.7f, -2.0f,  1.0f,     0.0f, 0.0f, 1.0f,	  0.3f, 0.0f,
								-1.0f, -2.0f,  1.0f,     0.0f, 0.0f, 1.0f,	  0.0f, 0.0f,
								-1.0f,  1.7f,  1.0f,     0.0f, 0.0f, 1.0f,	  0.0f, 1.0f,

								//Front Left Leg - Right
								-0.7f,  1.7f,  1.0f,     1.0f, 0.0f, 0.0f,	  0.0f, 1.0f,
								-0.7f,  1.7f,  0.7f,     1.0f, 0.0f, 0.0f,	  0.3f, 1.0f,
								-0.7f, -2.0f,  0.7f,     1.0f, 0.0f, 0.0f,	  0.3f, 0.0f,
								-0.7f, -2.0f,  0.7f,     1.0f, 0.0f, 0.0f,	  0.3f, 0.0f,
								-0.7f, -2.0f,  1.0f,     1.0f, 0.0f, 0.0f,	  0.0f, 0.0f,
								-0.7f,  1.7f,  1.0f,     1.0f, 0.0f, 0.0f,	  0.0f, 1.0f,

								//Front Left Leg - Back
								-0.7f,  1.7f,  0.7f,     0.0f, 0.0f, -1.0f,	  0.0f, 1.0f,
								-1.0f,  1.7f,  0.7f,     0.0f, 0.0f, -1.0f,	  0.3f, 1.0f,
								-1.0f, -2.0f,  0.7f,     0.0f, 0.0f, -1.0f,	  0.3f, 0.0f,
								-1.0f, -2.0f,  0.7f,     0.0f, 0.0f, -1.0f,	  0.3f, 0.0f,
								-0.7f, -2.0f,  0.7f,     0.0f, 0.0f, -1.0f,	  0.0f, 0.0f,
							    -0.7f,  1.7f,  0.7f,     0.0f, 0.0f, -1.0f,	  0.0f, 1.0f,

								//Front Left Leg - Left
								-1.0f,  1.7f,  0.7f,     -1.0f, 0.0f, 0.0f,	  0.0f, 1.0f,
								-1.0f,  1.7f,  1.0f,     -1.0f, 0.0f, 0.0f,	  0.3f, 1.0f,
								-1.0f, -2.0f,  1.0f,     -1.0f, 0.0f, 0.0f,	  0.3f, 0.0f,
								-1.0f, -2.0f,  1.0f,     -1.0f, 0.0f, 0.0f,	  0.3f, 0.0f,
								-1.0f, -2.0f,  0.7f,     -1.0f, 0.0f, 0.0f,	  0.0f, 0.0f,
								-1.0f,  1.7f,  0.7f,     -1.0f, 0.0f, 0.0f,	  0.0f, 1.0f,

								//Back Left Leg - Top
								-0.7f,  1.7f, -0.7f,     0.0f, 1.0f, 0.0f,    1.0f, 0.0f,   //12
								-0.7f,  1.7f, -1.0f,     0.0f, 1.0f, 0.0f,    1.0f, 1.0f,   //11
								-1.0f,  1.7f, -1.0f,     0.0f, 1.0f, 0.0f,    0.0f, 1.0f,   //6
								-1.0f,  1.7f, -1.0f,     0.0f, 1.0f, 0.0f,	  0.0f, 1.0f,
								-1.0f,  1.7f, -0.7f,     0.0f, 1.0f, 0.0f,    0.0f, 0.0f,   //13
								-0.7f,  1.7f, -0.7f,     0.0f, 1.0f, 0.0f,	  1.0f, 0.0f,

								//Back Left Leg - Bottom
								-0.7f, -2.0f, -0.7f,     0.0f, -1.0f, 0.0f,   1.0f, 0.0f,   //25
								-0.7f, -2.0f, -1.0f,     0.0f, -1.0f, 0.0f,   1.0f, 1.0f,   //24
								-1.0f, -2.0f, -1.0f,     0.0f, -1.0f, 0.0f,   0.0f, 1.0f,   //27
								-1.0f, -2.0f, -1.0f,     0.0f, -1.0f, 0.0f,   0.0f, 1.0f,
								-1.0f, -2.0f, -0.7f,     0.0f, -1.0f, 0.0f,   0.0f, 0.0f,   //26
								-0.7f, -2.0f, -0.7f,     0.0f, -1.0f, 0.0f,	  1.0f, 0.0f,

								//Back Left Leg - Front
								-1.0f,  1.7f, -0.7f,     0.0f, 0.0f, 1.0f,	  0.0f, 1.0f,
								-0.7f,  1.7f, -0.7f,     0.0f, 0.0f, 1.0f,	  0.3f, 1.0f,
								-0.7f, -2.0f, -0.7f,     0.0f, 0.0f, 1.0f,	  0.3f, 0.0f,
								-0.7f, -2.0f, -0.7f,     0.0f, 0.0f, 1.0f,	  0.3f, 0.0f,
								-1.0f, -2.0f, -0.7f,     0.0f, 0.0f, 1.0f,	  0.0f, 0.0f,
								-1.0f,  1.7f, -0.7f,     0.0f, 0.0f, 1.0f,	  0.0f, 1.0f,

								//Back Left Leg - Right
								-0.7f,  1.7f, -0.7f,     1.0f, 0.0f, 0.0f,	  0.0f, 1.0f,
								-0.7f,  1.7f, -1.0f,     1.0f, 0.0f, 0.0f,	  0.3f, 1.0f,
								-0.7f, -2.0f, -1.0f,     1.0f, 0.0f, 0.0f,	  0.3f, 0.0f,
								-0.7f, -2.0f, -1.0f,     1.0f, 0.0f, 0.0f,	  0.3f, 0.0f,
								-0.7f, -2.0f, -0.7f,     1.0f, 0.0f, 0.0f,	  0.0f, 0.0f,
								-0.7f,  1.7f, -0.7f,     1.0f, 0.0f, 0.0f,	  0.0f, 1.0f,

								//Back Left Leg - Back
								-0.7f,  1.7f, -1.0f,     0.0f, 0.0f, -1.0f,	  0.0f, 1.0f,
								-1.0f,  1.7f, -1.0f,     0.0f, 0.0f, -1.0f,	  0.3f, 1.0f,
								-1.0f, -2.0f, -1.0f,     0.0f, 0.0f, -1.0f,	  0.3f, 0.0f,
								-1.0f, -2.0f, -1.0f,     0.0f, 0.0f, -1.0f,	  0.3f, 0.0f,
								-0.7f, -2.0f, -1.0f,     0.0f, 0.0f, -1.0f,	  0.0f, 0.0f,
								-0.7f,  1.7f, -1.0f,     0.0f, 0.0f, -1.0f,	  0.0f, 1.0f,

								//Back Left Leg - Left
								-1.0f,  1.7f, -1.0f,     -1.0f, 0.0f, 0.0f,	  0.0f, 1.0f,
								-1.0f,  1.7f, -0.7f,     -1.0f, 0.0f, 0.0f,	  0.3f, 1.0f,
								-1.0f, -2.0f, -0.7f,     -1.0f, 0.0f, 0.0f,	  0.3f, 0.0f,
								-1.0f, -2.0f, -0.7f,     -1.0f, 0.0f, 0.0f,	  0.3f, 0.0f,
								-1.0f, -2.0f, -1.0f,     -1.0f, 0.0f, 0.0f,	  0.0f, 0.0f,
								-1.0f,  1.7f, -1.0f,     -1.0f, 0.0f, 0.0f,	  0.0f, 1.0f,

								//Seat - Top
								-1.3f,  2.3f,  1.3f,     0.0f, 1.0f, 0.0f,    0.0f, 0.0f,   //39
								 1.3f,  2.3f,  1.3f,     0.0f, 1.0f, 0.0f,    1.0f, 0.0f,   //36
								 1.3f,  2.3f, -1.3f,     0.0f, 1.0f, 0.0f,    1.0f, 1.0f,   //37
								 1.3f,  2.3f, -1.3f,     0.0f, 1.0f, 0.0f,	  1.0f, 1.0f,	//37
								-1.3f,  2.3f, -1.3f,     0.0f, 1.0f, 0.0f,    0.0f, 1.0f,   //38
								-1.3f,  2.3f,  1.3f,     0.0f, 1.0f, 0.0f,	  0.0f, 0.0f,	//39

								 //Seat - Bottom
								 1.3f,  2.0f,  1.3f,     0.0f, -1.0f, 0.0f,   1.0f, 0.0f,   //40
								 1.3f,  2.0f, -1.3f,     0.0f, -1.0f, 0.0f,   1.0f, 1.0f,   //41
								-1.3f,  2.0f, -1.3f,     0.0f, -1.0f, 0.0f,   0.0f, 1.0f,   //42
								-1.3f,  2.0f, -1.3f,     0.0f, -1.0f, 0.0f,	  0.0f, 1.0f,
								-1.3f,  2.0f,  1.3f,     0.0f, -1.0f, 0.0f,   0.0f, 0.0f,   //43
								 1.3f,  2.0f,  1.3f,     0.0f, -1.0f, 0.0f,	  1.0f, 0.0f,

								 //Seat - Front
								-1.3f,  2.3f,  1.3f,     0.0f, 0.0f, 1.0f,	  0.0f, 0.3f,
								 1.3f,  2.3f,  1.3f,     0.0f, 0.0f, 1.0f,	  1.0f, 0.3f,
								 1.3f,  2.0f,  1.3f,     0.0f, 0.0f, 1.0f,	  1.0f, 0.0f,
								 1.3f,  2.0f,  1.3f,     0.0f, 0.0f, 1.0f,	  1.0f, 0.0f,
								-1.3f,  2.0f,  1.3f,     0.0f, 0.0f, 1.0f,	  0.0f, 0.0f,
								-1.3f,  2.3f,  1.3f,     0.0f, 0.0f, 1.0f,	  0.0f, 0.3f,

								//Seat - Right
								 1.3f,  2.3f,  1.3f,     1.0f, 0.0f, 0.0f,	  0.0f, 0.3f,
								 1.3f,  2.3f, -1.3f,     1.0f, 0.0f, 0.0f,	  1.0f, 0.3f,
								 1.3f,  2.0f, -1.3f,     1.0f, 0.0f, 0.0f,	  1.0f, 0.0f,
								 1.3f,  2.0f, -1.3f,     1.0f, 0.0f, 0.0f,	  1.0f, 0.0f,
								 1.3f,  2.0f,  1.3f,     1.0f, 0.0f, 0.0f,	  0.0f, 0.0f,
								 1.3f,  2.3f,  1.3f,     1.0f, 0.0f, 0.0f,	  0.0f, 0.3f,

								 //Seat - Back
								 1.3f,  2.3f, -1.3f,     0.0f, 0.0f, -1.0f,	  0.0f, 0.3f,
								-1.3f,  2.3f, -1.3f,     0.0f, 0.0f, -1.0f,	  1.0f, 0.3f,
								-1.3f,  2.0f, -1.3f,     0.0f, 0.0f, -1.0f,	  1.0f, 0.0f,
								-1.3f,  2.0f, -1.3f,     0.0f, 0.0f, -1.0f,	  1.0f, 0.0f,
								 1.3f,  2.0f, -1.3f,     0.0f, 0.0f, -1.0f,	  0.0f, 0.0f,
								 1.3f,  2.3f, -1.3f,     0.0f, 0.0f, -1.0f,	  0.0f, 0.3f,

								 //Seat - left
								-1.3f,  2.3f, -1.3f,     -1.0f, 0.0f, 0.0f,	  1.0f, 0.0f,
								-1.3f,  2.3f,  1.3f,     -1.0f, 0.0f, 0.0f,	  0.0f, 0.0f,
								-1.3f,  2.0f,  1.3f,     -1.0f, 0.0f, 0.0f,	  0.0f, 0.3f,
								-1.3f,  2.0f,  1.3f,     -1.0f, 0.0f, 0.0f,	  1.0f, 0.0f,
								-1.3f,  2.0f, -1.3f,     -1.0f, 0.0f, 0.0f,	  0.0f, 0.0f,
								-1.3f,  2.3f, -1.3f,     -1.0f, 0.0f, 0.0f,	  0.0f, 0.3f,

								//ADDITION: Crossmember between back legs - Top
								-0.7f, -0.5f, -1.0f,      0.0f, 1.0f, 0.0f,	  1.0f, 0.0f,
								 0.7f, -0.5f, -1.0f,      0.0f, 1.0f, 0.0f,	  0.0f, 0.0f,
								 0.7f, -0.5f, -0.7f,      0.0f, 1.0f, 0.0f,	  0.0f, 0.3f,
								 0.7f, -0.5f, -0.7f,      0.0f, 1.0f, 0.0f,	  1.0f, 0.0f,
								-0.7f, -0.5f, -0.7f,      0.0f, 1.0f, 0.0f,	  0.0f, 0.0f,
								-0.7f, -0.5f, -1.0f,      0.0f, 1.0f, 0.0f,	  0.0f, 0.3f,

								//ADDITION: Crossmember between back legs - Bottom
								-0.7f, -1.0f, -1.0f,      0.0f, -1.0f, 0.0f,	  1.0f, 0.0f,
								 0.7f, -1.0f, -1.0f,      0.0f, -1.0f, 0.0f,	  0.0f, 0.0f,
								 0.7f, -1.0f, -0.7f,      0.0f, -1.0f, 0.0f,	  0.0f, 0.3f,
								 0.7f, -1.0f, -0.7f,      0.0f, -1.0f, 0.0f,	  1.0f, 0.0f,
								-0.7f, -1.0f, -0.7f,      0.0f, -1.0f, 0.0f,	  0.0f, 0.0f,
								-0.7f, -1.0f, -1.0f,      0.0f, -1.0f, 0.0f,	  0.0f, 0.3f,

								//ADDITION: Crossmember between back legs - Back
								-0.7f, -0.5f, -1.0f,      0.0f, 0.0f, -1.0f,	  1.0f, 0.0f,
								 0.7f, -0.5f, -1.0f,      0.0f, 0.0f, -1.0f,	  0.0f, 0.0f,
								 0.7f, -1.0f, -1.0f,      0.0f, 0.0f, -1.0f,	  0.0f, 0.3f,
								 0.7f, -1.0f, -1.0f,      0.0f, 0.0f, -1.0f,	  1.0f, 0.0f,
								-0.7f, -1.0f, -1.0f,      0.0f, 0.0f, -1.0f,	  0.0f, 0.0f,
								-0.7f, -0.5f, -1.0f,      0.0f, 0.0f, -1.0f,	  0.0f, 0.3f,

								//ADDITION: Crossmember between back legs - Front
								-0.7f, -0.5f, -0.7f,      0.0f, 0.0f, 1.0f,	  1.0f, 0.0f,
								 0.7f, -0.5f, -0.7f,      0.0f, 0.0f, 1.0f,	  0.0f, 0.0f,
								 0.7f, -1.0f, -0.7f,      0.0f, 0.0f, 1.0f,	  0.0f, 0.3f,
								 0.7f, -1.0f, -0.7f,      0.0f, 0.0f, 1.0f,	  1.0f, 0.0f,
								-0.7f, -1.0f, -0.7f,      0.0f, 0.0f, 1.0f,	  0.0f, 0.0f,
								-0.7f, -0.5f, -0.7f,      0.0f, 0.0f, 1.0f,	  0.0f, 0.3f,

								//ADDITION: Crossmember between front legs - Top
								-0.7f, -0.5f,  0.7f,      0.0f, 1.0f, 0.0f,	  1.0f, 0.0f,
								 0.7f, -0.5f,  0.7f,      0.0f, 1.0f, 0.0f,	  0.0f, 0.0f,
								 0.7f, -0.5f,  1.0f,      0.0f, 1.0f, 0.0f,	  0.0f, 0.3f,
								 0.7f, -0.5f,  1.0f,      0.0f, 1.0f, 0.0f,	  1.0f, 0.0f,
								-0.7f, -0.5f,  1.0f,      0.0f, 1.0f, 0.0f,	  0.0f, 0.0f,
								-0.7f, -0.5f,  0.7f,      0.0f, 1.0f, 0.0f,	  0.0f, 0.3f,

								//ADDITION: Crossmember between front legs - Bottom
								-0.7f, -1.0f,  0.7f,      0.0f, -1.0f, 0.0f,	  1.0f, 0.0f,
								 0.7f, -1.0f,  0.7f,      0.0f, -1.0f, 0.0f,	  0.0f, 0.0f,
								 0.7f, -1.0f,  1.0f,      0.0f, -1.0f, 0.0f,	  0.0f, 0.3f,
								 0.7f, -1.0f,  1.0f,      0.0f, -1.0f, 0.0f,	  1.0f, 0.0f,
								-0.7f, -1.0f,  1.0f,      0.0f, -1.0f, 0.0f,	  0.0f, 0.0f,
								-0.7f, -1.0f,  0.7f,      0.0f, -1.0f, 0.0f,	  0.0f, 0.3f,

								//ADDITION: Crossmember between front legs - Back
								-0.7f, -0.5f,  0.7f,      0.0f, 0.0f, -1.0f,	  1.0f, 0.0f,
								 0.7f, -0.5f,  0.7f,      0.0f, 0.0f, -1.0f,	  0.0f, 0.0f,
								 0.7f, -1.0f,  0.7f,      0.0f, 0.0f, -1.0f,	  0.0f, 0.3f,
								 0.7f, -1.0f,  0.7f,      0.0f, 0.0f, -1.0f,	  1.0f, 0.0f,
								-0.7f, -1.0f,  0.7f,      0.0f, 0.0f, -1.0f,	  0.0f, 0.0f,
								-0.7f, -0.5f,  0.7f,      0.0f, 0.0f, -1.0f,	  0.0f, 0.3f,

								//ADDITION: Crossmember between front legs - Front
								-0.7f, -0.5f,  1.0f,      0.0f, 0.0f, 1.0f,	  1.0f, 0.0f,
								 0.7f, -0.5f,  1.0f,      0.0f, 0.0f, 1.0f,	  0.0f, 0.0f,
								 0.7f, -1.0f,  1.0f,      0.0f, 0.0f, 1.0f,	  0.0f, 0.3f,
								 0.7f, -1.0f,  1.0f,      0.0f, 0.0f, 1.0f,	  1.0f, 0.0f,
								-0.7f, -1.0f,  1.0f,      0.0f, 0.0f, 1.0f,	  0.0f, 0.0f,
								-0.7f, -0.5f,  1.0f,      0.0f, 0.0f, 1.0f,	  0.0f, 0.3f,

								//ADDITION: Middle Crossmember - Top
								-0.1f, -0.5f, -0.7f,      0.0f, 1.0f, 0.0f,	  1.0f, 0.0f,
								 0.1f, -0.5f, -0.7f,      0.0f, 1.0f, 0.0f,	  0.0f, 0.0f,
								 0.1f, -0.5f,  0.7f,      0.0f, 1.0f, 0.0f,	  0.0f, 0.3f,
								 0.1f, -0.5f,  0.7f,      0.0f, 1.0f, 0.0f,	  1.0f, 0.0f,
								-0.1f, -0.5f,  0.7f,      0.0f, 1.0f, 0.0f,	  0.0f, 0.0f,
								-0.1f, -0.5f, -0.7f,      0.0f, 1.0f, 0.0f,	  0.0f, 0.3f,

								//ADDITION: Middle Crossmember - Bottom
								-0.1f, -1.0f, -0.7f,      0.0f, -1.0f, 0.0f,	  1.0f, 0.0f,
								 0.1f, -1.0f, -0.7f,      0.0f, -1.0f, 0.0f,	  0.0f, 0.0f,
								 0.1f, -1.0f,  0.7f,      0.0f, -1.0f, 0.0f,	  0.0f, 0.3f,
								 0.1f, -1.0f,  0.7f,      0.0f, -1.0f, 0.0f,	  1.0f, 0.0f,
								-0.1f, -1.0f,  0.7f,      0.0f, -1.0f, 0.0f,	  0.0f, 0.0f,
								-0.1f, -1.0f, -0.7f,      0.0f, -1.0f, 0.0f,	  0.0f, 0.3f,

								//ADDITION: Middle Crossmember - Right
								 0.1f, -0.5f, -0.7f,      1.0f, 0.0f, 0.0f,	  1.0f, 0.0f,
								 0.1f, -1.0f, -0.7f,      1.0f, 0.0f, 0.0f,	  0.0f, 0.0f,
								 0.1f, -1.0f,  0.7f,      1.0f, 0.0f, 0.0f,	  0.0f, 0.3f,
								 0.1f, -1.0f,  0.7f,      1.0f, 0.0f, 0.0f,	  1.0f, 0.0f,
								 0.1f, -0.5f,  0.7f,      1.0f, 0.0f, 0.0f,	  0.0f, 0.0f,
								 0.1f, -0.5f, -0.7f,      1.0f, 0.0f, 0.0f,	  0.0f, 0.3f,

								//ADDITION: Middle Crossmember - Left
								-0.1f, -1.0f, -0.7f,      -1.0f, 0.0f, 0.0f,	  1.0f, 0.0f,
								-0.1f, -0.5f, -0.7f,      -1.0f, 0.0f, 0.0f,	  0.0f, 0.0f,
								-0.1f, -0.5f,  0.7f,      -1.0f, 0.0f, 0.0f,	  0.0f, 0.3f,
								-0.1f, -0.5f,  0.7f,      -1.0f, 0.0f, 0.0f,	  1.0f, 0.0f,
								-0.1f, -1.0f,  0.7f,      -1.0f, 0.0f, 0.0f,	  0.0f, 0.0f,
								-0.1f, -1.0f, -0.7f,      -1.0f, 0.0f, 0.0f,	  0.0f, 0.3f,


							};

    //Generate buffer ids
    glGenVertexArrays(1, &StoolVAO);
    glGenBuffers(1, &VBO);

    //Activate the StoolVAO before binding and setting VBOs and VAPs
    glBindVertexArray(StoolVAO);

    //Activate the VBO
    glBindBuffer(GL_ARRAY_BUFFER, VBO);
    glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW); //Copy vertices to VBO

    //Set attribute pointer 0 to hold position data
    glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 8 * sizeof(GLfloat), (GLvoid*)0);
    glEnableVertexAttribArray(0); //Enables vertex attribute

    //Set attribute pointer 1 to hold Normal data
    glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, 8 * sizeof(GLfloat), (GLvoid*)(3 * sizeof(GLfloat)));
    glEnableVertexAttribArray(1);

    //Set attribute pointer 2 to hold Texture coordinate data
    glVertexAttribPointer(2, 2, GL_FLOAT, GL_FALSE, 8 * sizeof(GLfloat), (GLvoid*)(6 * sizeof(GLfloat)));
    glEnableVertexAttribArray(2);
    glBindVertexArray(0); //Unbind the stool VAO

    //Generate buffer ids for lamp (smaller stool)
    glGenVertexArrays(1, &LightVAO); //Vertex Array for stool vertex copies to serve as light source

    //Activate the Vertex Array Object before binding and setting any VBOs and Vertex Attribute Pointers
    glBindVertexArray(LightVAO);

    //Referencing the same VBO for its vertices
    glBindBuffer(GL_ARRAY_BUFFER, VBO);

    //Set attribute pointer to 0 to hold Position data (used for the lamp)
    glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 8 * sizeof(GLfloat), (GLvoid*)0);
    glEnableVertexAttribArray(0);
    glBindVertexArray(0);

}

/*Generate and load the texture*/
void UGenerateTexture(){
    glGenTextures(1, &texture);

    glBindTexture(GL_TEXTURE_2D, texture);

    int width, height;

    unsigned char* image = SOIL_load_image("dRYh8r.jpg", &width, &height, 0, SOIL_LOAD_RGB); //Loads texture file **REVISED TO A LIGHT GRAY WOOD GRAIN IMAGE

    glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB, width, height, 0, GL_RGB, GL_UNSIGNED_BYTE, image);

    glGenerateMipmap(GL_TEXTURE_2D);

    SOIL_free_image_data(image);

    glBindTexture(GL_TEXTURE_2D, 0); //Unbind the texture

}

/*Implements the UKeyboard function*/
void UKeyboard(unsigned char key, GLint x, GLint y)
{
	switch(key){
					case 'o':
						currentKey = key;
						cout<<"You pressed W!"<<endl;
						break;

					default:
						cout<<"Press a key!"<<endl;
	}
}

/*Implements the UKeyReleased function*/
void UKeyReleased(unsigned char key, GLint x, GLint y)
{
			cout<<"Key released!"<<endl;
			currentKey = '0';
}


/*Implements the UMouseMove function*/
void UMouseMove(int x, int y)
{
	//Immediately replaces center locked coordinates with new mouse coordinates
	if(mouseDetected)
	{
		lastMouseX = x;
		lastMouseY = y;
		mouseDetected = false;
	}

	//Gets the direction the mouse was moved in x and y
	mouseXOffset = x - lastMouseX;
	mouseYOffset = lastMouseY - y;   //Inverted Y

	//Updates with new mouse coordinates
	lastMouseX = x;
	lastMouseY = y;

	//Applies sensitivity to mouse direction
	mouseXOffset *= sensitivity;
	mouseYOffset *= sensitivity;

	//Accumulates the yaw and pitch variables
	yaw += mouseXOffset;
	pitch += mouseYOffset;

	if(pitch > 89.0f)
		pitch = 89.0f;
	if(pitch < -89.0f)
		pitch = -89.0f;

	//Orbits around the center
	front.x = 10.0f * cos(yaw);
	front.y = 10.0f * sin(pitch);
	front.z = sin(yaw) * cos(pitch) * 10.0f;
}



