---
title: "Building a Recipe Generator with Generative AI"
date: 2025-01-09T11:32:00+05:45
slug: building-a-recipe-generator-with-generative-ai
aliases:
  - /blogs/building-a-recipe-generator-with-generative-AI/
  - /blogs/building-a-recipe-generator-with-generative-ai/
  - /blogs/Generate-Recipes-from-Ingredients-in-Image/
categories:
  - AI
  - Generative AI
summary: "How I built an AI-powered tool to generate recipes from fridge photos using Gemini, RAG, and structured outputs."
description: "Discover how generative AI can turn random fridge ingredients into creative recipes. Learn about image analysis, structured outputs, and retrieval-augmented generation."
cover:
   image: "https://cdn.rishavdahal.com.np/recipe-generator-ai.jpg"
   alt: "AI-generated recipe generator analyzing kitchen ingredients"
   caption: "Intelligent recipe generator powered by Gemini and Multimodal AI"
   relative: false
showtoc: true
draft: false
tags: ["AI", "recipes", "machine learning", "Gemini", "RAG"]
author: "Rishav Dahal"
keywords: ["recipe generator", "AI", "fridge ingredients", "Gemini", "RAG", "structured outputs"]
---

## From Fridge to Feast
> *How I used Gemini, RAG, and structured outputs to solve my "What's for dinner?" problem.*

### The Problem: Kitchen Chaos
We’ve all been there: staring into a fridge full of random ingredients—half a bell pepper, leftover chicken, wilting spinach—and wondering, “What can I even make with this?” Recipe websites rarely help because they assume you have pantry staples or time for grocery runs.

### My Goal: Build an AI-powered tool that:
- Identifies ingredients from text or fridge photos.
- Generates creative recipes using only those ingredients.
- Suggests similar recipes for inspiration.

### Enter Generative AI—specifically, Google’s Gemini.

### How Generative AI Solves This
I combined three key Gen AI capabilities:

1. **Image Understanding: “What’s in my fridge?”**  
   Using Gemini’s vision model, the app analyzes uploaded photos to detect ingredients. For example:

   ```python
   def analyze_image(image_path):
       model = genai.GenerativeModel('gemini-pro-vision')
       response = model.generate_content(["List ingredients in this image:", Image.open(image_path)])
       return response.text

   # Detects: "chicken, rice, broccoli, soy sauce"
   ```

   This lets users snap a fridge photo instead of typing ingredients—perfect for rushed weeknights.

2. **Structured Outputs: Recipes in JSON**  
   Recipes need consistency. Using few-shot prompting, I guided Gemini to output recipes in a JSON format:

   ```python
   prompt = f"""
   Generate a recipe using ONLY: {ingredients}.  
   Use this JSON structure:  
   {examples}  # Few-shot examples here
   """

   # Example Output:
   {
     "name": "Garlic Butter Chicken & Rice",
     "ingredients": ["chicken", "rice", "garlic"],
     "steps": ["Sauté garlic", "Cook chicken", "Mix with rice"]
   }
   ```

   Structured outputs make recipes machine-readable for apps or meal planners.

3. **Retrieval-Augmented Generation (RAG): “Inspire Me!”**  
   To avoid repetitive recipes, I stored 500+ recipes in a ChromaDB vector database. When a user asks for "quick dinners," RAG retrieves similar recipes to inspire Gemini:

   ```python
   def retrieve_similar_recipes(query):
       results = db.query(query_texts=[query], n_results=2)
       return results["documents"]  # Returns: ["Teriyaki Chicken Bowl", "Vegetable Fried Rice"]
   ```

   This ensures variety while keeping ingredients relevant.

### Challenges & Limitations
- **Vision Model Accuracy**: Gemini sometimes misidentifies uncommon ingredients (e.g., “kale” vs. “spinach”).
- **Unrealistic Combinations**: The AI occasionally suggests odd pairings (like “broccoli smoothies”).
- **Scalability**: ChromaDB works for small datasets, but larger apps need managed vector databases.

### Future Ideas
- **Dietary Filters**: “Make this gluten-free!”
- **Smart Fridge Integration**: Auto-detect expiring ingredients.
- **Taste Profiles**: “Generate a spicy version.”

### Try It Yourself
Explore the full code in my [Kaggle Notebook](https://www.kaggle.com/code/risaavdahal/generate-recipes-from-ingredients). Next time you’re stuck with random ingredients, let Gen AI handle the meal planning!

### Conclusion
Generative AI isn’t just for chatbots—it can solve everyday problems like kitchen chaos. By combining vision models, structured outputs, and RAG, we can build tools that are both creative and practical. What will you cook up next?
```