# Overview

## Concept

The concept is to create a website which has a bio/profile and two tools. First tool is a job fit interface which allows pasting of a job description into a query that generates an evaluation of job fit. This form or website would interface at the back end with an LLM with a closed, persistent context that consists of my resume, experiences, significant projects, education, certifications, and other material that I have produced.
The second tool would be a virtual interview that works off the same persistant LLM context, The HR person would type in or paste interview questions for pre-qualification, and the AI would generate answers based on the context it has been provided, which is my resume, experience, education and other materials that I’ve produced.

## Technical Implementation

For technical implementation, the key problem to solve is how do we persist a context within an LLM and use it as a backend for the website
