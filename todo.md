# Convex Agents Project Todos

## Recipe Pantry Assistant

An AI-powered cooking assistant where users snap photos of their fridge/pantry, and the agent suggests recipes they can make with what they have.

### Core User Flow
1. **Upload** → Snap a photo of your fridge/pantry
2. **Analyze** → AI identifies available ingredients from the image
3. **Search** → AI searches recipe database for matches
4. **Suggest** → Returns 3-5 recipes ranked by "ingredients you have" vs "need to buy"
5. **Act** → View full recipe, save favorites, or generate shopping list

### Features Used from Convex Agents

| Feature | Implementation |
|---------|---------------|
| **Files/Images** | Upload pantry photos; AI vision model extracts ingredients |
| **Tools** | 3 tools: `searchRecipes`, `saveFavorite`, `generateShoppingList` |
| **RAG** | Recipe database with vector embeddings for semantic search |
| **Structured Output** | Recipe suggestions as `{name, ingredients, steps, difficulty, missingCount}` |
| **Threads** | Each cooking session is a thread; remembers dietary restrictions |
| **Context Search** | Search past favorite recipes for inspiration |
| **Streaming** | Stream recipe generation with "thinking" status |
| **Human Agents** | Option to request help from a human chef for substitutions |

### Agent Design

```typescript
const chefAgent = new Agent(components.agent, {
  name: "Pantry Chef",
  languageModel: openai.chat("gpt-4o"), // Vision-capable
  instructions: `You are a helpful chef. When given pantry photos:
    1. Identify all visible ingredients
    2. Use searchRecipes to find matches
    3. Prioritize recipes using ingredients they have
    4. Be mindful of dietary restrictions from thread context`,
  tools: {
    searchRecipes,      // RAG search on recipe DB
    saveFavorite,       // Save to user's favorites
    generateShoppingList, // Create list of missing ingredients
  },
  textEmbeddingModel: openai.embedding("text-embedding-3-small"),
});
```

### Tool Details

**`searchRecipes`** - Uses RAG component
- Args: `query` (what user wants), `availableIngredients[]`, `maxMissing` (tolerance)
- Searches recipe embeddings for matches
- Filters by dietary tags (vegan, gluten-free, etc.)
- Returns top matches with similarity scores

**`saveFavorite`** - Convex mutation via tool
- Args: `recipeId`, `notes`
- Saves to user's profile for later
- Can be referenced in future threads via context search

**`generateShoppingList`** - Structured output
- Args: `recipeId`, `availableIngredients[]`
- Returns: `ShoppingList { items[], estimatedCost?, stores? }`

### Data Model

```typescript
// Recipes (populated via RAG ingestion)
recipes: {
  name: string,
  ingredients: string[],
  instructions: string[],
  tags: string[], // "vegan", "quick", "italian"
  embeddings: vector[],
}

// User favorites
favorites: {
  userId: string,
  recipeId: string,
  notes: string,
  cookedCount: number,
}

// Shopping lists
shoppingLists: {
  userId: string,
  items: { ingredient: string, checked: boolean }[],
  fromRecipeId: string,
}
```

### UI Components

1. **Camera Upload** - Snap pantry photo
2. **Ingredient Chips** - Editable list of detected items
3. **Recipe Cards** - Show match %, missing ingredients count, time
4. **Thread View** - Chat-style back-and-forth ("I have eggs too now")
5. **Favorites Page** - Past saved recipes with search
6. **Shopping List** - Check off items as you buy

### Workflow Example

```typescript
// In an action
export const suggestRecipes = action({
  handler: async (ctx, { imageFileId, threadId }) => {
    const { thread } = await chefAgent.continueThread(ctx, { threadId });

    // 1. Vision analysis + ingredient extraction
    const analysis = await thread.generateText({
      message: {
        role: "user",
        content: [
          { type: "image", fileId: imageFileId },
          { type: "text", text: "What ingredients do you see?" }
        ]
      }
    });

    // 2. Search recipes (agent decides to call tool)
    const result = await thread.generateText({
      prompt: "Suggest 3 recipes based on these ingredients",
      stopWhen: stepCountIs(3), // Allow tool calls
    });

    // Messages saved automatically to thread
  }
});
```

### Bonus Features to Add Later

- **Meal Planning Workflow** - Multi-day workflow planning meals, generating consolidated shopping lists
- **Cooking Mode** - Step-by-step with timer tools
- **Image Recognition for Cooked Meals** - Rate your attempt, agent gives feedback
- **Social Sharing** - Share recipes with friends (human agents)

---

## Implementation Tasks

- [x] Initialize Convex project with Agent component
- [ ] Set up RAG component for recipe database
- [ ] Create recipe data ingestion pipeline
- [ ] Build `chefAgent` with tools
- [ ] Create image upload handler
- [ ] Build React UI components
- [ ] Add streaming for recipe generation
- [ ] Implement favorites and shopping list features
- [ ] usage tracking
