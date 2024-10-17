```dataviewjs
// Replace with your actual OpenRouter.ai API key
const OPENROUTER_API_KEY = 'API_KEY_HERE'; // Replace with your API key

// Define the topics, their specific instructions, and callout types
const TOPICS = [
    {
        name: 'The Universe',
        tag: 'Universe',
        instructions: `
# INSTRUCTIONS
1. Research the most recent and significant events in space exploration, astronomy, or cosmology.
2. Focus on new discoveries, space missions, or major theories in astrophysics.
3. Prioritize updates from space agencies (NASA, SpaceX, ESA) or prominent scientific research findings.
4. Ensure the information covers any technological advancements or notable space phenomena.
        `,
        calloutType: 'info' // Supported types: info, warning, tip, success, question, etc.
    },
    {
        name: 'The Planet',
        tag: 'Planet',
        instructions: `
# INSTRUCTIONS
1. Investigate current events related to climate change, environmental policy, and sustainability efforts.
2. Look for news on natural disasters, conservation efforts, and shifts in renewable energy adoption.
3. Highlight any governmental actions or international agreements aimed at protecting the environment.
4. Prioritize updates that impact ecosystems, biodiversity, or global sustainability initiatives.
        `,
        calloutType: 'danger'
    },
    {
        name: 'Culture',
        tag: 'Culture',
        instructions: `
# INSTRUCTIONS
1. Research the latest trends in arts, entertainment, fashion, and social movements around the world.
2. Include stories about cultural shifts, new forms of artistic expression, and major global or local cultural events.
3. Focus on social movements that are reshaping societal values (e.g., gender equality, racial justice, LGBTQ+ rights).
4. Highlight any significant developments in global pop culture or shifts in consumer behavior that reflect cultural changes.
        `,
        calloutType: 'tip'
    },
    {
        name: 'Politics',
        tag: 'Politics',
        instructions: `
# INSTRUCTIONS
1. Investigate recent political developments, including new laws, government policies, and diplomatic relations.
2. Focus on major elections, international negotiations, or conflicts that are shaping global or national political landscapes.
3. Prioritize any changes in governance, human rights, or civil liberties that affect populations.
4. Include news about political figures, parties, and power dynamics influencing national or global policies.
        `,
        calloutType: 'warning'
    },
    {
        name: 'Economics',
        tag: 'Economics',
        instructions: `
# INSTRUCTIONS
1. Research current global or national economic trends, focusing on inflation, market movements, and economic policies.
2. Include updates on financial markets, cryptocurrency developments, and changes in consumer behavior or business innovation.
3. Investigate major shifts in employment, trade relations, and corporate mergers or industry disruptions.
4. Prioritize any economic policies that impact daily living costs, wages, or business conditions.
        `,
        calloutType: 'success'
    },
    {
        name: 'Science',
        tag: 'Science',
        instructions: `
# INSTRUCTIONS
1. Investigate the latest breakthroughs in scientific research across fields like biology, physics, chemistry, and medicine.
2. Highlight new discoveries or theories that are reshaping our understanding of the natural world.
3. Include significant developments in medical research, including new treatments, vaccines, or health initiatives.
4. Prioritize findings from prominent research institutions or scientific journals.
        `,
        calloutType: 'danger'
    },
    {
        name: 'Technology',
        tag: 'Technology',
        instructions: `
# INSTRUCTIONS
1. Research the latest developments in consumer technology, artificial intelligence, cybersecurity, and data privacy.
2. Include updates on emerging technologies such as blockchain, quantum computing, and advancements in AI.
3. Focus on innovations in tech that are reshaping industries or daily life, including new gadgets, apps, or digital services.
4. Highlight any major announcements from key tech companies and changes in tech policies or regulations.
        `,
        calloutType: 'bug'
    }
];

// Prompt template for fetching news
const PROMPT_TEMPLATE = `
Hey **Curator 📚**! I need your help compiling a structured update on the latest about **{{topic}}** as of **{{date}}**. Your responsibility is to perform an Advanced Search to ensure the information is up-to-date, accurate, and properly formatted according to the provided schema, and optimized for an Obsidian markdown note.

# Schema
**Overview**:
A brief summary of the Daily News of **{{topic}}**

**News**
Bulleted summaries of the latest news on **{{topic}}** as of **{{date}}**, following the instructions provided.

**Citations**
A list of markdown hyperlinked sources [title](www.websitelink.com)

**Suggested Notes**
- [[Related Note 1]]
- [[Related Note 2]]
- [[Related Note ...]]

{{instructions}}

You must provide fair citations and specific evidence for every piece of news.
`;

// Function to fetch news for a given topic
async function fetchNews(topic, instructions, date) {
    const prompt = PROMPT_TEMPLATE
        .replace(/{{topic}}/g, topic)
        .replace('{{date}}', date)
        .replace('{{instructions}}', instructions);

    console.log(`\n---\nFetching news for topic: "${topic}" on ${date}`);
    console.log(`Prompt:\n${prompt}\n`);

    try {
        const response = await fetch("https://openrouter.ai/api/v1/chat/completions", {
            method: "POST",
            headers: {
                "Authorization": `Bearer ${OPENROUTER_API_KEY}`,
                "Content-Type": "application/json"
            },
            body: JSON.stringify({
                "model": "perplexity/llama-3.1-sonar-large-128k-online",
                "messages": [
                    { "role": "user", "content": prompt },
                ],
                "temperature": 0
            })
        });

        console.log(`API request sent for topic "${topic}". Status: ${response.status}`);

        if (!response.ok) {
            const errorText = await response.text();
            throw new Error(`API request failed with status ${response.status}: ${errorText}`);
        }

        let data;
        try {
            data = await response.json();
        } catch (jsonError) {
            throw new Error(`Failed to parse JSON response for topic "${topic}": ${jsonError.message}`);
        }

        const content = data.choices?.[0]?.message?.content?.trim() || 'No content generated.';
        console.log(`Received content for topic "${topic}": ${content.substring(0, 100)}...`);
        return content;

    } catch (error) {
        console.error(`Error fetching news for topic "${topic}":`, error);
        return `Error fetching news for this topic. Please check the console for more details.`;
    }
}

// Function to create the daily news note
async function createDailyNewsNote() {
    const today = window.moment().format('YYYY-MM-DD');
    const noteTitle = `${today} News`;

    // Get the active file (parent note)
    const parentFile = app.workspace.getActiveFile();
    if (!parentFile) {
        console.error('No active file found.');
        new Notice('📰: No active file found.');
        return;
    }

    const folderPath = parentFile.parent ? parentFile.parent.path : '';
    const filePath = folderPath ? `${folderPath}/${noteTitle}.md` : `${noteTitle}.md`;

    console.log(`\n---\nCreating daily news note: "${noteTitle}" in folder: "${folderPath || 'root'}"`);

    // Check if the note already exists
    const existingFile = app.vault.getAbstractFileByPath(filePath);
    if (existingFile) {
        console.warn(`The note "${noteTitle}" already exists.`);
        new Notice(`The note "${noteTitle}" already exists.`);
        return;
    }

    // Initialize the front matter
    let frontMatter = `---
title: ${noteTitle}
tags: dailyNews`;

    // Collect tags from topics
    const topicTags = TOPICS.map(topic => topic.tag);
    if (topicTags.length > 0) {
        frontMatter += `, ${topicTags.join(', ')}`;
    }
    frontMatter += `
---
\n`;

    // Fetch news for all topics in parallel
    const newsPromises = TOPICS.map(topicObj => fetchNews(topicObj.name, topicObj.instructions, today));

    console.log('Fetching news for all topics in parallel...');
    const newsContents = await Promise.all(newsPromises);
    console.log('All news fetched.');

    // Initialize the content of the note
    let noteContent = frontMatter;

    // Append each topic's news within a callout block
    TOPICS.forEach((topicObj, index) => {
        const { name: topic, calloutType } = topicObj;
        const newsContent = newsContents[index];

        console.log(`Appending news for topic "${topic}" with callout type "${calloutType}"`);

        // Format the content within a collapsed callout block
        // Using "-" after the callout type makes it collapsed by default
        noteContent += `> [!${calloutType}]- ${topic}\n`;
        // Indent the news content with ">" for blockquote
        const indentedContent = newsContent.split('\n').map(line => `> ${line}`).join('\n');
        noteContent += `${indentedContent}\n\n`;
    });

    console.log('All topics appended to the note content.');

    // Create the new note
    try {
        await app.vault.create(filePath, noteContent);
        console.log(`Daily news note "${noteTitle}" created successfully at "${filePath}".`);
        new Notice(`Daily news note "${noteTitle}" has been created.`);
    } catch (creationError) {
        console.error(`Error creating the note "${noteTitle}":`, creationError);
        new Notice(`Error creating the note "${noteTitle}". Check the console for details.`);
    }
}

// Function to render the dailyNews button
function renderDailyNewsButton() {
    console.log('Rendering the "Generate Daily News" button.');

    // Create a container div
    const container = document.createElement('div');
    container.style.margin = '10px 0';

    // Create the button
    const button = document.createElement('button');
    button.textContent = '📰 Generate Daily News';
    button.style.padding = '10px 20px';
    button.style.backgroundColor = '#007ACC';
    button.style.color = 'white';
    button.style.border = 'none';
    button.style.borderRadius = '5px';
    button.style.cursor = 'pointer';
    button.style.fontSize = '16px';

    // Optional: Add hover effect
    button.onmouseover = () => {
        button.style.backgroundColor = '#005A9E';
    };
    button.onmouseout = () => {
        button.style.backgroundColor = '#007ACC';
    };

    // Handle button click
    button.onclick = async () => {
        button.disabled = true;
        const originalText = button.textContent;
        button.textContent = 'Generating...';
        console.log('Button clicked: Starting generation process.');
        new Notice('📰: Starting the generation process...');

        try {
            await createDailyNewsNote();
        } catch (error) {
            console.error('Error generating daily news:', error);
            new Notice('📰: An error occurred while generating the daily news.');
        } finally {
            button.textContent = originalText;
            button.disabled = false;
            console.log('Generation process completed.');
        }
    };

    // Append button to the container
    container.appendChild(button);

    // Append the container to the Dataview API's container
    dv.container.appendChild(container);
    console.log('"Generate Daily News" button rendered successfully.');
}

// Render the button when the script runs
renderDailyNewsButton();
```
