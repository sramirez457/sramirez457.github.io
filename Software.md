---
layout: default
title: "Software Engineering & Database"
permalink: /software/
---
[home](https://sramirez457.github.io/)


## Software Engineering and Database
**_Demonstrate an ability to use well-founded and innovative techniques, skills, and tools in computing practices for the purpose of implementing computer solutions that deliver value and accomplish industry-specific goals._**

# Software Design/Engineering
Enhancement One focused on Software Design and Enginering. The artifact I chose for Enhancement One was my final project from CS330, which involved the creation of a 3D representation of a stool. I created this artifact two semesters ago and it was my first project that used 3D rendering. I created it using C++ in OpenGL in Eclipse. Even though I completed the assignment as instructed, I felt there was room for improvement. 
   
I selected this item because it is a good representation of something unique that I worked on during my coursework. This artifact showcases my development skills in my creation of my code, as well as the design and development of my specific stool. I developed a code that created my stool shape, gave it texture and light, and allowed orbiting by mouse movement. My originally submitted stool was basic and looked slightly different than my original image. With this in mind, I was able to change a few portions on my code to better represent the original stool. My improvements included changing the background color, the stool texture, and the color of the light. I also slowed the camera speed and mouse sensitivity, so it made it easier to orbit. Finally, I added three extra cross members to the bottom half of the stool. By making these enhancements, I met the course objectives I planned to in Module One.  Although the stool still does not match perfectly to the original, I think it has been improved. 
   
During my enhancement process, I realized that I had not utilized a majority of the skills I had acquired during my CS330 coursework. Since it was only two semesters ago that I developed this code, I had not had the opportunity to create any additional 3D rendering since. Taking this into account, I had to look up a couple of details that I had forgotten. For example, I had to look up the color combinations when changing the background. I also had to look up what “normals” associated with my vertices meant and how to determine their values. I also had to get back into the mindset of creating a 3D image on paper and assigning points on the x, y, and z planes. Furthermore, I have not used C++ or OpenGL since this course either. I think that this a challenge I have faced before since our coursework is diverse and includes different languages applied in various ways. Apart from these small challenges, I think that my Enhancement One was successful in improving my artifact. 

See complete Enhanced Stool code [here](https://sramirez457.github.io/software/stool/).

| Original Stool | Enhanced Stool |
| --- | ---|
| <img width="801" alt="OrigStool" src="https://user-images.githubusercontent.com/73710194/102022966-e6debd80-3d4f-11eb-868a-6c43c3abefa7.png"> | <img width="799" alt="Enhanced Stool" src="https://user-images.githubusercontent.com/73710194/102022968-ea724480-3d4f-11eb-9775-0c201e4423ff.png"> |

# Databases
Enhancement Three focused on Databases. The artifact I chose for Enhancement Three was the same as the one for Enhancement Two, my final project from CS340, which I created last semester. My final project was to create a reporting service for basic stock market securities information using MongoDB and the RESTful API web-based protocol. For Enhancement Three, I chose to focus on the following two prompts of the project directly related to databases: 
   
1.	Assess the need for indexing as you formulate queries and, using the MongoDB shell, create any needed single or compound indexes. 
2.	In your RESTful API, enable the following functionality (advanced querying), ensuring your code is functional, reusable, concise, and commented: Select and present specific stock summary information by a user-derived lit of ticker symbols. 

I justified the inclusion of this artifact because MongoDB is a document-oriented database, that provides an alternative to the widely used MySQL. This artifact displayed my abilities beyond the basic create, read, update, and delete skill set. I showed my ability to analyze queries and chose the most appropriate indexes that would be the most efficient. Originally, I thought that creating a compound index would be more efficient than a single index. But I found that creating a more appropriate single index was most time efficient. I created a single index that decreased from 15 to 4 millis for a search for Profit Margin between 0.1 and 0.2, sorted by Ticker. When working on the RESTful API portion, I wanted to create a more efficient function to query the database for the ticker symbol list. I think that I developed a better alternative to my original code. I believe I met the course objective that I planned for my enhancements. 
   
While working on these enhancements, I realized that a concept may not work as well as originally thought during planning. During my work with the indexes, I tried the same query in three different situations: (1) Ticker Index, (2) Compound Index with Ticker and Profit Margin, and (3) Profit Margin. I had to delete and create indexes for these three different situations to figure out which was the most effective in millis (see screenshots below). In the end, the best outcome was not what I had originally planned but I learned from the process. 

| Ticker Index | 
| --- |
| <img width="654" alt="Default Index" src="https://user-images.githubusercontent.com/73710194/102024517-142f6980-3d58-11eb-899b-a7e6ed56d05f.png"> | 

| Compound Index |
| --- |
| <img width="594" alt="Compound Index" src="https://user-images.githubusercontent.com/73710194/102024521-17c2f080-3d58-11eb-8f9b-bbd840ed72ed.png"> | 

| Profit Margin Index |
| --- |
| <img width="609" alt="Profit Index" src="https://user-images.githubusercontent.com/73710194/102024523-1a254a80-3d58-11eb-934e-7ff0e96f7091.png"> |

| Original stockReport | 
| --- | 
| <img width="667" alt="Original stockReport" src="https://user-images.githubusercontent.com/73710194/102022912-7f287280-3d4f-11eb-8d90-bcb1004f4dc7.png"> | 

| Enhanced stockReport | 
| --- |
| <img width="575" alt="New stockReport" src="https://user-images.githubusercontent.com/73710194/102022915-82236300-3d4f-11eb-9ef7-04ed94c4aec3.png"> |



[home](https://sramirez457.github.io/)
