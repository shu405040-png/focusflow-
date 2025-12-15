# focusflow-
简约风的单词记忆卡片


import React, { useState, useEffect, useRef } from 'react';
import { Plus, Play, Check, X, RotateCcw, Trash2, BookOpen, Brain, ChevronRight, Volume2, Sparkles } from 'lucide-react';

// --- Firebase and API Setup (Required for Canvas Environment) ---
const apiKey = ""; // Canvas environment provides this if empty
const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};

// Utility for exponential backoff retry logic
const withBackoff = async (fn, maxRetries = 5) => {
    for (let i = 0; i < maxRetries; i++) {
        try {
            return await fn();
        } catch (error) {
            if (i === maxRetries - 1) {
                console.error("Max retries reached. Failing.", error);
                throw error;
            }
            const delay = Math.pow(2, i) * 1000 + Math.random() * 1000;
            console.warn(`Retry ${i + 1} in ${delay.toFixed(0)}ms...`);
            await new Promise(resolve => setTimeout(resolve, delay));
        }
    }
};

// Function to call Gemini for text generation (Example Sentence)
const callGeminiApi = async (prompt) => {
    const apiUrl = `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`;
    const payload = {
        contents: [{ parts: [{ text: prompt }] }],
        systemInstruction: {
            parts: [{ text: "You are a language teacher assistant. Provide only a single, concise, and natural English sentence as an example for the user's word. Do not include any introductory phrases or quotation marks." }]
        },
        tools: [{ "google_search": {} }], // Use grounding for relevant context
    };

    return withBackoff(async () => {
        const response = await fetch(apiUrl, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(payload)
        });

        if (!response.ok) throw new Error(`API error: ${response.statusText}`);

        const result = await response.json();
        return result.candidates?.[0]?.content?.parts?.[0]?.text || 'Failed to generate example sentence.';
    });
};

// --- TTS Utility Functions ---

const base64ToArrayBuffer = (base64) => {
    const binaryString = atob(base64);
    const len = binaryString.length;
    const bytes = new Uint8Array(len);
    for (let i = 0; i < len; i++) {
        bytes[i] = binaryString.charCodeAt(i);
    }
    return bytes.buffer;
};

const writeString = (view, offset, string) => {
    for (let i = 0; i < string.length; i++) {
        view.setUint8(offset + i, string.charCodeAt(i));
    }
};

const pcmToWav = (pcm16, sampleRate = 24000) => {
    const numChannels = 1;
    const bytesPerSample = 2; // Int16
    const blockAlign = numChannels * bytesPerSample;
    const byteRate = sampleRate * blockAlign;
    const dataSize = pcm16.byteLength;
    const buffer = new ArrayBuffer(44 + dataSize);
    const view = new DataView(buffer);

    // RIFF header
    writeString(view, 0, 'RIFF');
    view.setUint32(4, 36 + dataSize, true);
    writeString(view, 8, 'WAVE');

    // fmt chunk
    writeString(view, 12, 'fmt ');
    view.setUint32(16, 16, true); // chunkSize
    view.setUint16(20, 1, true); // wFormatTag (1 for PCM)
    view.setUint16(22, numChannels, true);
    view.setUint32(24, sampleRate, true);
    view.setUint32(28, byteRate, true);
    view.setUint16(32, blockAlign, true);
    view.setUint16(34, 16, true); // bitsPerSample

    // data chunk
    writeString(view, 36, 'data');
    view.setUint32(40, dataSize, true);

    // Write PCM data
    let offset = 44;
    for (let i = 0; i < pcm16.length; i++) {
        view.setInt16(offset, pcm16[i], true);
        offset += bytesPerSample;
    }

    return new Blob([view], { type: 'audio/wav' });
};

const callTtsApi = async (text) => {
    const apiUrl = `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-tts:generateContent?key=${apiKey}`;

    const payload = {
        contents: [{ parts: [{ text: `Say clearly: ${text}` }] }],
        generationConfig: {
            responseModalities: ["AUDIO"],
            speechConfig: {
                voiceConfig: { prebuiltVoiceConfig: { voiceName: "Zephyr" } }
            }
        },
    };

    return withBackoff(async () => {
        const response = await fetch(apiUrl, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(payload)
        });

        if (!response.ok) throw new Error(`TTS API error: ${response.statusText}`);

        const result = await response.json();
        const part = result?.candidates?.[0]?.content?.parts?.[0];
        
        if (part?.inlineData?.data && part?.inlineData?.mimeType.startsWith("audio/L16;")) {
            const mimeType = part.inlineData.mimeType;
            const sampleRateMatch = mimeType.match(/rate=(\d+)/);
            const sampleRate = sampleRateMatch ? parseInt(sampleRateMatch[1], 10) : 24000;
            
            const pcmData = base64ToArrayBuffer(part.inlineData.data);
            const pcm16 = new Int16Array(pcmData);
            const wavBlob = pcmToWav(pcm16, sampleRate);
            return URL.createObjectURL(wavBlob);
        }
        throw new Error('TTS response missing valid audio data.');
    });
};


const FocusFlow = () => {
  // --- State Management ---
  // 所有词汇已根据你的要求导入，并自动生成了例句。
  const initialCards = [
    { id: 1, en: 'Like a moth to flame', cn: '就像飞蛾扑火', status: 'new', example: 'She was drawn to the glamorous city lights like a moth to a flame.' },
    { id: 2, en: 'Irresistible', cn: '不可抗拒的', status: 'new', example: 'The aroma of freshly baked bread was utterly irresistible.' },
    { id: 3, en: 'Inexorable', cn: '不可抗拒的', status: 'new', example: 'The inexorable march of time affects everyone.' },
    { id: 4, en: 'Inexorably', cn: '不可抗拒地', status: 'new', example: 'Technology is inexorably changing the way we live.' },
    { id: 5, en: "But that's not exactly what's going on", cn: '但事情并不完全是这样', status: 'new', example: "She thought he was joking, but that's not exactly what was going on." },
    { id: 6, en: 'alcohol', cn: '酒精', status: 'new', example: 'He avoids drinking alcohol for health reasons.' },
    { id: 7, en: 'scramble', cn: '扰乱，弄乱', status: 'new', example: 'The children managed to scramble the entire puzzle box in minutes.' },
    { id: 8, en: 'innate', cn: '天生的', status: 'new', example: 'She has an innate ability to understand complex mathematics.' },
    { id: 9, en: 'Sign off', cn: '暂停工作', status: 'new', example: 'I need to sign off this project by the end of the day.' },
    { id: 10, en: 'Struggle with', cn: '做什么很艰难', status: 'new', example: 'Many students struggle with advanced calculus.' },
    { id: 11, en: 'The rise of', cn: '的兴起', status: 'new', example: 'The rise of social media transformed public communication.' },
    { id: 12, en: 'Lag', cn: '落后 赶不上', status: 'new', example: 'My internet connection started to lag during the video call.' },
    { id: 13, en: 'fitted', cn: '合身的', status: 'new', example: 'She looked elegant in her neatly fitted suit.' },
    { id: 14, en: 'vibrant', cn: '充满活力的，鲜艳的', status: 'new', example: 'The city has a vibrant nightlife and cultural scene.' },
    { id: 15, en: 'Suburb', cn: '郊外', status: 'new', example: 'They decided to move from the city center to a quiet suburb.' },
    { id: 16, en: 'Faculty', cn: '全体教师', status: 'new', example: 'The entire faculty attended the graduation ceremony.' },
    { id: 17, en: 'Cultivate', cn: '培养', status: 'new', example: 'It is important to cultivate good habits early in life.' },
    { id: 18, en: 'Unleash', cn: '释放', status: 'new', example: 'The strong winds unleashed a torrent of leaves and debris.' },
    { id: 19, en: 'Surge', cn: '激增', status: 'new', example: 'We saw a sudden surge in online traffic after the announcement.' },
    { id: 20, en: 'Optimize', cn: '优化', status: 'new', example: "Developers are working to optimize the app's performance." },
    { id: 21, en: 'Upcoming', cn: '即将到来的', status: 'new', example: 'The upcoming concert is already sold out.' },
    { id: 22, en: 'congratulatory', cn: '祝贺的', status: 'new', example: 'He received a flood of congratulatory messages after his win.' },
    { id: 23, en: 'Term', cn: '术语', status: 'new', example: 'Please define the key terms used in this contract.' },
    { id: 24, en: 'Expedition', cn: '远征，考察', status: 'new', example: 'They embarked on a scientific expedition to the Antarctic.' },
    { id: 25, en: 'collaborative', cn: '合作的', status: 'new', example: 'The project requires a highly collaborative effort from all teams.' },
    { id: 26, en: 'captured', cn: '捕获的', status: 'new', example: 'The journalist captured the chaos of the event in a single photograph.' },
    { id: 27, en: 'Verify', cn: '核实，查证', status: 'new', example: 'You must verify the authenticity of the documents before submission.' },
    { id: 28, en: 'Reveal', cn: '揭示，透露', status: 'new', example: 'The study results reveal a surprising trend in consumer behavior.' },
    { id: 29, en: 'Notable', cn: '值得注意的', status: 'new', example: 'Her contributions to the field of quantum physics are highly notable.' },
    { id: 30, en: 'Retail', cn: '零售', status: 'new', example: 'Many traditional retail stores are moving towards online sales.' },
    { id: 31, en: 'Exceeded', cn: '超越的', status: 'new', example: 'The final cost far exceeded our initial budget estimation.' },
    { id: 32, en: 'Robust', cn: '稳固的，强健的', status: 'new', example: 'We need a robust security system to protect our data.' },
    { id: 33, en: 'Sentiment', cn: '情绪，观点', status: 'new', example: 'The general sentiment among the employees was positive.' },
    { id: 34, en: 'Attribute', cn: '归因于', status: 'new', example: 'She attributes her success to hard work and dedication.' },
    { id: 35, en: 'Occurrence', cn: '发生，事件', status: 'new', example: 'Power outages are a rare occurrence in this neighborhood.' },
    { id: 36, en: 'peak travel', cn: '旅游高峰期', status: 'new', example: 'We try to avoid flying during peak travel season.' },
    { id: 37, en: 'Anticipate', cn: '预期，预料', status: 'new', example: 'We anticipate that the project will be completed next month.' },
    { id: 38, en: 'Capacity', cn: '容纳能力', status: 'new', example: 'The stadium has a seating capacity of 50,000 people.' },
    { id: 39, en: 'Take selfies', cn: '自拍', status: 'new', example: 'Tourists love to take selfies with the famous monuments.' },
    { id: 40, en: 'Bund', cn: '码头，堤岸', status: 'new', example: 'The historic buildings along the Bund in Shanghai are famous.' },
    { id: 41, en: 'Megacity', cn: '大城市', status: 'new', example: "Tokyo is one of the world's largest megacities." },
    { id: 42, en: 'Whopping', cn: '巨大的', status: 'new', example: 'They ordered a whopping twelve pizzas for the party.' },
    { id: 43, en: 'Whop', cn: '猛击', status: 'new', example: 'The sound of the ball hitting the ground was a loud whop.' },
    { id: 44, en: 'Box office', cn: '票房收入', status: 'new', example: 'The movie broke all records at the box office.' },
    { id: 45, en: 'Surpass', cn: '超过，胜过', status: 'new', example: 'His early work was good, but his later novels surpass it.' },
    { id: 46, en: 'Surpassing', cn: '卓越的', status: 'new', example: 'The beauty of the landscape was of a surpassing quality.' },
    { id: 47, en: 'Dominating', cn: '支配的，主要的', status: 'new', example: 'He has a dominating presence in the business world.' },
    { id: 48, en: 'Gross', cn: '总共的，毛的', status: 'new', example: "The company's gross income increased by 15% this quarter." },
    { id: 49, en: 'Eclipse', cn: '使黯然失色', status: 'new', example: "The new singer's popularity quickly eclipsed that of the established star." },
    { id: 50, en: 'Milestone', cn: '里程碑', status: 'new', example: 'Graduating from university was a major milestone in her life.' },
    { id: 51, en: 'Get underway', cn: '开始发生', status: 'new', example: 'The project is scheduled to get underway next week.' },
    { id: 52, en: 'Vitality', cn: '活力，生机', status: 'new', example: 'Regular exercise is essential for maintaining vitality.' },
    { id: 53, en: 'Distribute', cn: '分发，分配', status: 'new', example: 'The organizers distributed pamphlets to everyone in attendance.' },
    { id: 54, en: 'Alter', cn: '改变，改动', status: 'new', example: 'You might need to alter your plans if the weather changes.' },
    { id: 55, en: 'Proceed', cn: '开始行动，开展', status: 'new', example: 'We can proceed with the next step of the experiment now.' },
    { id: 56, en: 'Consecutive', cn: '连续的', status: 'new', example: 'The team won five consecutive games this season.' },
    { id: 57, en: 'Once in a while', cn: '偶尔', status: 'new', example: 'I only see my distant cousin once in a while.' },
    { id: 58, en: 'Draft', cn: '草稿，草案', status: 'new', example: 'Please submit the first draft of your essay by Friday.' },
    { id: 59, en: 'Resolution', cn: '决议，决心', status: 'new', example: "She made a New Year's resolution to learn a new skill." },
    { id: 60, en: 'revolution', cn: '革命', status: 'new', example: 'The industrial revolution changed society profoundly.' },
    { id: 61, en: 'cultural rights', cn: '文化权利', status: 'new', example: 'The organization advocates for the protection of cultural rights.' },
    { id: 62, en: 'within the context of', cn: '在……的背景下', status: 'new', example: 'We must analyze the decision within the context of history.' },
    { id: 63, en: 'Address', cn: '处理，设法解决', status: 'new', example: 'The committee will address the issue of funding next week.' },
    { id: 64, en: 'Vote', cn: '投票', status: 'new', example: 'Every citizen has the right to vote in the election.' },
    { id: 65, en: 'on behalf of', cn: '代表', status: 'new', example: 'I am speaking on behalf of my entire family.' },
    { id: 66, en: 'Ambassador', cn: '大使', status: 'new', example: 'She served as an ambassador to the United Nations.' },
    { id: 67, en: 'anniversary', cn: '周年纪念日', status: 'new', example: 'They celebrated their tenth wedding anniversary last week.' },
    { id: 68, en: 'Statement', cn: '声明，报告', status: 'new', example: 'The company released a public statement regarding the merger.' },
    { id: 69, en: 'Declaration', cn: '报告，声明', status: 'new', example: 'The government issued a declaration of emergency.' },
    { id: 70, en: 'Thematic', cn: '主题的', status: 'new', example: 'The film explores deep thematic elements of loss and memory.' },
    { id: 71, en: 'Interactive', cn: '交互式的', status: 'new', example: 'The museum features many interactive exhibits for children.' },
    { id: 72, en: 'Dialogue', cn: '对话，交换意见', status: 'new', example: 'Effective communication requires open dialogue between parties.' },
    { id: 73, en: 'Effectively', cn: '有效地', status: 'new', example: 'She effectively managed the team and finished the project on time.' },
    { id: 74, en: 'Invest', cn: '投资', status: 'new', example: 'It is wise to invest in your long-term education.' },
    { id: 75, en: 'Investigate', cn: '调查，研究', status: 'new', example: 'The police were called in to investigate the strange occurrence.' },
    { id: 76, en: 'Commend', cn: '赞扬，赞许', status: 'new', example: 'The manager was commended for his excellent leadership.' },
    { id: 77, en: 'Recommend', cn: '建议，推荐', status: 'new', example: 'Can you recommend a good book to read during the flight?' },
    { id: 78, en: 'Extent', cn: '程度，范围', status: 'new', example: 'To what extent do you agree with this policy change?' },
    { id: 79, en: 'Session', cn: '会议', status: 'new', example: 'The legislative session will resume tomorrow morning.' },
    { id: 80, en: 'Mission', cn: '使命，任务', status: 'new', example: 'His personal mission was to help the less fortunate.' },
    { id: 81, en: 'Admit', cn: '承认', status: 'new', example: 'She reluctantly admitted that she had made a mistake.' },
    { id: 82, en: 'Raise', cn: '提高，提起', status: 'new', example: 'The government plans to raise taxes on luxury goods.' },
    { id: 83, en: 'Raised', cn: '凸起的', status: 'new', example: 'There was a small, raised bump on his forehead.' },
    { id: 84, en: 'A sort of', cn: '一种', status: 'new', example: 'It was a sort of informal meeting, not a formal conference.' },
    { id: 85, en: 'Ensconce', cn: '安置，安顿下来', status: 'new', example: 'He ensconced himself comfortably in the armchair with a book.' },
    { id: 86, en: 'Rather', cn: '宁愿，最好', status: 'new', example: 'I would rather walk than take the bus today.' },
    { id: 87, en: 'Stout', cn: '壮实的，坚固的', status: 'new', example: 'The old oak table was very stout and reliable.' },
    { id: 88, en: 'Benevolent', cn: '仁慈的', status: 'new', example: 'The benevolent king was loved by all his subjects.' },
    { id: 89, en: 'in spite of', cn: '尽管', status: 'new', example: 'They went ahead with the picnic in spite of the rain.' },
    { id: 90, en: 'Tushe', cn: '长牙，獠牙', status: 'new', example: 'The wild boar sharpened its tushes on the tree bark.' },
    { id: 91, en: 'Before long', cn: '不久之后', status: 'new', example: 'Before long, the little seedlings will grow into tall trees.' },
    { id: 92, en: 'Settle down', cn: '安定下来', status: 'new', example: 'After moving, it took her a few weeks to settle down in the new city.' },
    { id: 93, en: 'Flutter up', cn: '鸟飞起来', status: 'new', example: 'A flock of pigeons began to flutter up from the park bench.' },
    { id: 94, en: 'beast', cn: '野兽', status: 'new', example: 'The forest was rumored to be home to a mythical beast.' },
    { id: 95, en: 'Vendor', cn: '供应商，小贩', status: 'new', example: 'The street vendor sold hot dogs and pretzels from his cart.' },
    { id: 96, en: 'Cornucopia', cn: '丰盛，丰富', status: 'new', example: 'The market stall offered a cornucopia of fresh fruits and vegetables.' },
    { id: 97, en: 'Get away with', cn: '做坏事而未受惩罚', status: 'new', example: "You can't just break the rules and expect to get away with it." },
    { id: 98, en: 'Captive', cn: '被限制的', status: 'new', example: 'The company holds a captive audience through its exclusive content.' },
    { id: 99, en: 'Severely', cn: '非常严重地', status: 'new', example: 'His work was severely criticized by the reviewers.' },
    { id: 100, en: 'Contract', cn: '合同，契约', status: 'new', example: 'They signed a long-term contract for the property lease.' },
    { id: 101, en: 'Literal', cn: '字面上的', status: 'new', example: 'You should not take his suggestion in the literal sense.' },
    { id: 102, en: 'Prisoner', cn: '囚犯，俘虏', status: 'new', example: 'The prisoner was granted parole after serving ten years.' },
    { id: 103, en: 'Poison', cn: '毒药', status: 'new', example: 'The detective suspected that the victim was killed by poison.' },
    { id: 104, en: 'Cafeteria', cn: '自助食堂', status: 'new', example: 'We usually eat lunch together in the office cafeteria.' },
    { id: 105, en: 'Evolve', cn: '进化，演变', status: 'new', example: 'The software continues to evolve with user feedback.' },
    { id: 106, en: 'Generic', cn: '通用的，无商标的', status: 'new', example: 'The store sells both brand-name and generic medicines.' },
    { id: 107, en: 'Term', cn: '期限，任期', status: 'new', example: 'The president will serve a four-year term in office.' },
    { id: 108, en: 'Inception', cn: '开端，创始', status: 'new', example: 'The company has grown significantly since its inception in 2005.' },
    { id: 109, en: 'Essentially', cn: '本质上', status: 'new', example: 'Essentially, the two proposals are the same.' },
    { id: 110, en: 'Essential', cn: '必不可少的', status: 'new', example: 'Water is absolutely essential for human survival.' },
    { id: 111, en: 'Iconic', cn: '象征性的', status: 'new', example: 'The Eiffel Tower is an iconic symbol of Paris.' },
    { id: 112, en: 'Fantastic', cn: '极好的', status: 'new', example: 'We had a fantastic time at the concert last night.' },
    { id: 113, en: 'Portable', cn: '便于携带的', status: 'new', example: 'He always carries a portable charger for his phone.' },
    { id: 114, en: 'Handy', cn: '好用的', status: 'new', example: 'Keep a flashlight handy in case of a power cut.' },
    { id: 115, en: 'Fragrance', cn: '香气，香水', status: 'new', example: 'She chose a light floral fragrance for the summer.' },
    { id: 116, en: 'streets and alleys', cn: '街巷', status: 'new', example: 'We spent the afternoon exploring the narrow streets and alleys of the old town.' },
    { id: 117, en: 'Stimulation', cn: '刺激，激励', status: 'new', example: 'Reading provides great mental stimulation for the mind.' },
    { id: 118, en: 'The tip of', cn: '某物的最末端', status: 'new', example: 'The cat sat right on the tip of the roof.' },
    { id: 119, en: 'The tip of the iceberg', cn: '冰山一角', status: 'new', example: 'The documented issues are just the tip of the iceberg.' },
    { id: 120, en: 'Transition', cn: '过渡，转变', status: 'new', example: 'The transition from college life to professional work was challenging.' },
    { id: 121, en: 'Terrain', cn: '地形，地势', status: 'new', example: 'The military vehicles are designed to handle rough terrain.' },
    { id: 122, en: 'Summarise', cn: '概括', status: 'new', example: 'Could you briefly summarise the main points of the presentation?' },
    { id: 123, en: 'Present', cn: '目前的', status: 'new', example: 'The challenges we face at present are quite significant.' },
    { id: 124, en: 'Landscape', cn: '风景，景色', status: 'new', example: 'The stunning mountainous landscape took my breath away.' },
    { id: 125, en: 'Destiny', cn: '命运', status: 'new', example: 'She believes that meeting him was a matter of destiny.' },
    { id: 126, en: 'Intertwine', cn: '相互交织', status: 'new', example: 'Their fates became intertwined after the accident.' },
    { id: 127, en: 'biosphere', cn: '生物圈', status: 'new', example: "Protecting the local biosphere is crucial for the planet's health." },
    { id: 128, en: 'Attractions', cn: '旅游胜地', status: 'new', example: 'London has many popular tourist attractions, like the Tower of London.' },
    { id: 129, en: 'Statue', cn: '雕像', status: 'new', example: 'The massive bronze statue overlooks the city harbor.' },
    { id: 130, en: 'Buddhist', cn: '佛教徒', status: 'new', example: 'He practices meditation as part of his Buddhist faith.' },
    { id: 131, en: 'Sentient', cn: '有感情的', status: 'new', example: 'We must consider the feelings of all sentient beings.' },
    { id: 132, en: 'renounce', cn: '宣布放弃', status: 'new', example: 'The prince decided to renounce his claim to the throne.' },
    { id: 133, en: 'Accomplishments', cn: '成就', status: 'new', example: 'Her academic accomplishments are truly remarkable.' },
    { id: 134, en: 'Shortcut', cn: '捷径', status: 'new', example: 'There is no easy shortcut to mastering a new language.' },
    { id: 135, en: 'Hardwired', cn: '与生俱来的', status: 'new', example: 'Humans are hardwired to seek social connection.' },
    { id: 136, en: 'to get a kick out of something', cn: '真正享受某事', status: 'new', example: 'He gets a kick out of watching action movies.' },
    { id: 137, en: 'To kick the habit', cn: '戒掉坏习惯', status: 'new', example: 'It took him months of effort to kick the smoking habit.' },
    { id: 138, en: 'Alive and kicking', cn: '健康活跃', status: 'new', example: 'Despite the rumors, the old band is still alive and kicking.' },
    { id: 139, en: 'To kick back', cn: '放松', status: 'new', example: 'After a long week, I just want to kick back and watch a movie.' },
    { id: 140, en: 'To kick the bucket', cn: '死掉', status: 'new', example: 'The old car finally kicked the bucket on the highway.' },
    { id: 141, en: 'To be kicked out', cn: '被赶出去', status: 'new', example: 'They were kicked out of the restaurant for being too loud.' },
    { id: 142, en: 'Have a kick to it', cn: '有劲儿/辣味', status: 'new', example: 'This curry is delicious, and it definitely has a kick to it.' },
    { id: 143, en: 'To kick butt', cn: '表现出色', status: 'new', example: 'Our team went out there and kicked butt in the championship game.' },
    { id: 144, en: 'To kick someone’s butt', cn: '击败对手', status: 'new', example: "If we practice hard, we will kick the opponent's butt in the next match." },
    { id: 145, en: 'To kick yourself', cn: '后悔', status: 'new', example: 'He will kick himself later for missing such an amazing opportunity.' },
    { id: 146, en: 'Scream', cn: '尖叫', status: 'new', example: 'The children screamed with excitement on the roller coaster.' },
    { id: 147, en: 'Kickstart', cn: '开始', status: 'new', example: 'The new policy aims to kickstart the local economy.' },
    { id: 148, en: 'Kick off', cn: '开始', status: 'new', example: 'The football match will kick off at 8 PM.' },
    { id: 149, en: 'Kickback', cn: '回扣', status: 'new', example: 'The scandal involved officials taking kickbacks from construction companies.' },
    { id: 150, en: 'Regulate', cn: '控制，管理', status: 'new', example: 'The government regulates the banking and finance industry.' },
    { id: 151, en: 'Vulnerable', cn: '脆弱的', status: 'new', example: 'Children are often the most vulnerable members of society.' },
    { id: 152, en: 'Scarce', cn: '缺乏的，不足的', status: 'new', example: 'Fresh water is a scarce resource in that desert region.' },
    { id: 153, en: 'Border', cn: '边界', status: 'new', example: 'The river forms a natural border between the two countries.' },
    { id: 154, en: 'Sacred', cn: '神圣的', status: 'new', example: 'The temple is considered a sacred place by many believers.' },
    { id: 155, en: 'District', cn: '区域，地区', status: 'new', example: 'They live in the historical district of the capital city.' },
    { id: 156, en: 'Remember (resemble)', cn: '看起来像，与什么相似', status: 'new', example: 'He has a face that reminds me of an old film star.' },
    { id: 157, en: 'Skyrocket', cn: '猛涨，飞涨', status: 'new', example: 'Housing prices in the city are expected to skyrocket next year.' },
    { id: 158, en: 'Assets', cn: '资产', status: 'new', example: "The company's total assets exceed fifty million dollars." },
    { id: 159, en: 'Precious', cn: '宝贵的，珍贵的', status: 'new', example: 'Time is our most precious and limited resource.' },
    { id: 160, en: 'Previous', cn: '以前的，先前的', status: 'new', example: 'The current system is much better than the previous one.' },
    { id: 161, en: 'Universally', cn: '普遍地', status: 'new', example: 'His work is universally recognized as groundbreaking.' },
    { id: 162, en: 'Tempered', cn: '温和的', status: 'new', example: 'His harsh critique was tempered by a few words of encouragement.' },
    { id: 163, en: 'Remark', cn: '评论，言论', status: 'new', example: 'His final remark was a thoughtful summary of the discussion.' },
    { id: 164, en: 'Beauty is in the eye of the beholder', cn: '情人眼里出西施', status: 'new', example: 'You may not like the painting, but remember, beauty is in the eye of the beholder.' },
    { id: 165, en: 'Dedicated', cn: '致力于', status: 'new', example: 'She is a dedicated teacher who works hard for her students.' },
    { id: 166, en: 'Exert', cn: '运用，施加', status: 'new', example: 'They exerted considerable pressure on the government to change the law.' },
    { id: 167, en: 'Resemble', cn: '像，看起来像', status: 'new', example: 'The new design closely resembles the original prototype.' },
    { id: 168, en: 'Astonished', cn: '震惊的', status: 'new', example: 'We were astonished by the sudden announcement of his retirement.' },
    { id: 169, en: 'ankle', cn: '脚踝', status: 'new', example: 'She twisted her ankle while hiking down the mountain trail.' },
    { id: 170, en: 'Flex', cn: '弯曲', status: 'new', example: 'The gymnast can flex her body into incredibly difficult positions.' },
    { id: 171, en: 'Flex bedroom', cn: '多功能卧室', status: 'new', example: 'The apartment features a small flex bedroom that can be used as an office.' },
    { id: 172, en: 'Controversial', cn: '有争议的', status: 'new', example: 'The decision to raise tuition fees was highly controversial.' },
    { id: 173, en: 'Processed', cn: '加工的', status: 'new', example: 'He tries to limit the amount of processed food in his diet.' },
    { id: 174, en: 'Whole food', cn: '天然食物', status: 'new', example: 'Eating whole foods contributes to better long-term health.' },
    { id: 175, en: 'Consultant', cn: '顾问', status: 'new', example: 'She hired a financial consultant to manage her investments.' },
    { id: 176, en: 'Penthouse', cn: '顶层公寓', status: 'new', example: 'They live in a luxurious penthouse apartment overlooking the harbor.' },
    { id: 177, en: 'Slang', cn: '俚语', status: 'new', example: 'Avoid using too much slang in formal academic writing.' },
    { id: 178, en: 'Criminal', cn: '罪犯', status: 'new', example: 'The police are actively searching for the escaped criminal.' },
    { id: 179, en: 'Scripted', cn: '照稿子念的', status: 'new', example: 'The presenter delivered a heavily scripted and dull speech.' },
    { id: 180, en: 'Comprehensive', cn: '综合的，全部的', status: 'new', example: 'The guide provides a comprehensive list of local restaurants.' },
    { id: 181, en: 'Racial', cn: '种族的', status: 'new', example: 'The community is working to eliminate racial inequality.' },
    { id: 182, en: 'in the presence of', cn: '在...面前', status: 'new', example: 'He was nervous about speaking in the presence of the CEO.' },
    { id: 183, en: 'Anticipated', cn: '预期的', status: 'new', example: 'The event concluded earlier than the anticipated time.' },
    { id: 184, en: 'Sector', cn: '领域', status: 'new', example: 'The finance sector saw rapid growth last quarter.' },
    { id: 185, en: 'Straightforward', cn: '坦率的，直率的', status: 'new', example: 'Give me a straightforward answer without any excuses.' },
    { id: 186, en: 'Insignificant', cn: '无足轻重的', status: 'new', example: 'The cost difference was statistically insignificant.' },
    { id: 187, en: 'Rival', cn: '竞争对手', status: 'new', example: 'The two companies are fierce rivals in the smartphone market.' },
    { id: 188, en: 'Sacrifice', cn: '牺牲', status: 'new', example: 'Many soldiers made the ultimate sacrifice for their country.' },
    { id: 189, en: 'Socialize', cn: '交往，交际', status: 'new', example: "It's important to socialize with your colleagues outside of work." },
    { id: 190, en: 'Implement', cn: '执行，贯彻', status: 'new', example: 'They decided to implement the new software immediately.' },
    { id: 191, en: 'Magnetic', cn: '有吸引力的', status: 'new', example: 'She has a magnetic personality that draws people to her.' },
    { id: 192, en: 'Go over', cn: '获得认可', status: 'new', example: 'The new policies must first go over well with the employees.' },
    { id: 193, en: 'Mysterious', cn: '神秘的', status: 'new', example: 'The ancient ruins are shrouded in a mysterious silence.' },
    { id: 194, en: 'Fundraiser', cn: '募捐活动', status: 'new', example: 'The annual gala is the biggest fundraiser for the charity.' },
    { id: 195, en: 'Persistent', cn: '持续的，坚持不懈的', status: 'new', example: 'She was persistent in her efforts to find a solution.' },
    { id: 196, en: 'Remain', cn: '保持不变，仍然是', status: 'new', example: 'The essential facts of the case remain unchanged.' },
    { id: 197, en: 'Source of', cn: '来源之处', status: 'new', example: 'Renewable energy is a key source of power for the future.' },
    { id: 198, en: 'Underscore', cn: '突出，凸显', status: 'new', example: 'The report underscores the need for better funding.' },
    { id: 199, en: 'Transaction', cn: '交易', status: 'new', example: 'The entire transaction was completed within minutes.' },
    { id: 200, en: 'Productive', cn: '富有成效的', status: 'new', example: 'We had a very productive meeting this morning.' },
    { id: 201, en: 'Expiration', cn: '到期，结束', status: 'new', example: 'Please check the expiration date on your passport.' },
    { id: 202, en: 'Expire', cn: '到期，失效', status: 'new', example: 'The license will automatically expire at the end of the year.' },
    { id: 203, en: 'Splurge', cn: '挥霍', status: 'new', example: 'She decided to splurge on a new designer handbag.' },
    { id: 204, en: 'Assembly', cn: '集会，装配', status: 'new', example: 'The car assembly line runs twenty-four hours a day.' },
    { id: 205, en: 'Durable', cn: '耐用的', status: 'new', example: 'This brand of luggage is known for being extremely durable.' },
    { id: 206, en: 'Grit', cn: '勇气，毅力', status: 'new', example: 'It took a lot of grit and determination to finish the marathon.' },
    { id: 207, en: 'Convey', cn: '表达，传递', status: 'new', example: 'Words alone cannot convey the depth of my gratitude.' },
    { id: 208, en: 'Mediocre', cn: '平庸的', status: 'new', example: 'His performance in the play was considered mediocre at best.' },
    { id: 209, en: 'Cord', cn: '粗线，电线', status: 'new', example: 'Be careful not to trip over the electrical cord.' },
    { id: 210, en: 'Brick', cn: '砖块', status: 'new', example: 'The house was built with traditional red clay bricks.' },
    { id: 211, en: 'Loyal', cn: '忠诚的', status: 'new', example: 'Dogs are famously loyal companions to humans.' },
    { id: 212, en: 'Apparently', cn: '显然地', status: 'new', example: 'Apparently, the meeting has been postponed until tomorrow.' },
    { id: 213, en: 'Imply', cn: '暗示', status: 'new', example: 'His silence seemed to imply agreement with the proposal.' },
    { id: 214, en: 'Concerning', cn: '关于', status: 'new', example: 'We need to discuss a few issues concerning the budget.' },
    { id: 215, en: 'it will awaken them to reality', cn: '这会让他们认清现实', status: 'new', example: 'The sudden loss of funding will awaken them to reality.' },
    { id: 216, en: 'Endeavor', cn: '努力，奋力', status: 'new', example: 'She will constantly endeavor to achieve her personal best.' },
    { id: 217, en: 'Hinder', cn: '阻碍，妨碍', status: 'new', example: 'Heavy snow may hinder rescue efforts in the remote areas.' },
    { id: 218, en: 'Entrepreneurial', cn: '具有企业家精神的', status: 'new', example: 'The city encourages entrepreneurial activity among young residents.' },
    { id: 219, en: 'Curse', cn: '咒语，祸根', status: 'new', example: 'The ancient tomb was said to be protected by a powerful curse.' },
    { id: 220, en: 'Cursed', cn: '被诅咒的', status: 'new', example: 'He felt like he was cursed with bad luck after the incident.' },
    { id: 221, en: 'Interpret', cn: '解释，说明', status: 'new', example: 'It is difficult to accurately interpret historical documents.' },
    { id: 222, en: 'Affection', cn: '喜爱', status: 'new', example: 'She showed great affection for her younger sister.' },
    { id: 223, en: 'Acquaintance', cn: '熟人', status: 'new', example: 'I saw an old acquaintance from college at the store.' },
    { id: 224, en: 'Intimate', cn: '亲密的', status: 'new', example: 'They shared a very intimate connection and mutual trust.' },
    { id: 225, en: 'Gaze', cn: '凝视', status: 'new', example: 'He continued to gaze out at the ocean horizon.' },
    { id: 226, en: 'Loyalty', cn: '忠诚', status: 'new', example: 'His loyalty to the team was never questioned.' },
    { id: 227, en: 'Attachment', cn: '依恋感', status: 'new', example: 'The child developed a strong emotional attachment to his teddy bear.' },
    { id: 228, en: 'Separate', cn: '分离，分开', status: 'new', example: 'The two rooms are separated by a thin wall.' },
    { id: 229, en: 'Instil', cn: '灌输，培养', status: 'new', example: 'Parents should instil good values in their children from an early age.' },
    { id: 230, en: 'Cultivate', cn: '培养', status: 'new', example: 'It takes time to cultivate a genuine friendship.' },
    { id: 231, en: 'Mike instilled the idea that all occupations are inferior except reading in his grandsons', cn: 'Mike灌输给他的孙子们，除了阅读之外，所有职业都是低人一等的想法', status: 'new', example: 'This phrase highlights the power of a specific belief being instilled in others.' },
    { id: 232, en: 'Firm', cn: '坚定的，牢固的', status: 'new', example: 'He took a firm stance against the proposed budget cuts.' },
    { id: 233, en: 'Reinforce', cn: '加强，强化', status: 'new', example: "The new evidence helped to reinforce the lawyer's argument." },
    { id: 234, en: 'Younger sibling', cn: '弟弟妹妹', status: 'new', example: 'My younger sibling is currently studying abroad for a semester.' },
    { id: 235, en: 'Tutor', cn: '导师，辅导', status: 'new', example: 'She hired a math tutor to help her prepare for the exam.' },
    { id: 236, en: 'Nap', cn: '小睡，打盹', status: 'new', example: 'I like to take a short nap after lunch on the weekends.' },
    { id: 237, en: 'Direct', cn: '径直的，直接的', status: 'new', example: 'We need to establish a direct line of communication with the client.' },
    { id: 238, en: 'Attributes', cn: '属性', status: 'new', example: 'Kindness is one of his most defining attributes.' },
    { id: 239, en: 'Stigma', cn: '耻辱，污名', status: 'new', example: 'There is still a social stigma attached to mental health issues.' },
    { id: 240, en: 'Enduring', cn: '持久的，持续的', status: 'new', example: 'She has a profound and enduring love for classical music.' },
    { id: 241, en: 'Obligation', cn: '义务，责任', status: 'new', example: 'He feels a strong obligation to help his aging parents.' },
    { id: 242, en: 'Paramount', cn: '至为重要的，首要的', status: 'new', example: 'Safety is of paramount importance in this laboratory.' },
    { id: 243, en: 'Divorce', cn: '离婚，分离', status: 'new', example: 'The sudden financial difficulties led to their divorce.' },
    { id: 244, en: 'Tempt', cn: '诱惑', status: 'new', example: 'The thought of quitting my job was tempting, but I resisted.' },
    { id: 245, en: 'Tempting', cn: '诱人的', status: 'new', example: 'The dessert menu looked incredibly tempting.' },
    { id: 246, en: 'Assist', cn: '帮助，协助', status: 'new', example: 'A good assistant can greatly assist the whole team.' },
    { id: 247, en: 'Eliminate', cn: '消除，淘汰', status: 'new', example: 'The company is trying to eliminate all unnecessary expenses.' },
    { id: 248, en: 'Erase', cn: '抹去，擦掉', status: 'new', example: 'He wished he could simply erase the memory of the argument.' },
    { id: 249, en: 'State', cn: '陈述，说明', status: 'new', example: 'Please state your name and address clearly for the record.' },
    { id: 250, en: 'Immature', cn: '不成熟的', status: 'new', example: 'His constant joking showed a lack of maturity.' },
    { id: 251, en: 'Amateur', cn: '业余爱好者', status: 'new', example: 'The painting competition is open to both professionals and amateurs.' },
    { id: 252, en: 'come away with', cn: '带走（感受，经验）', status: 'new', example: 'I came away from the seminar with a new understanding of the topic.' },
    { id: 253, en: 'Recognize', cn: '承认，识别', status: 'new', example: 'I didn\'t recognize her at first because she had changed her hairstyle.' },
    { id: 254, en: 'Olive', cn: '橄榄', status: 'new', example: 'They grow olives and produce high-quality olive oil.' },
    { id: 255, en: 'Punctually', cn: '守时地', status: 'new', example: 'She always arrives punctually for her morning shift.' },
    { id: 256, en: 'Cereal', cn: '谷物', status: 'new', example: 'I usually eat a bowl of cold cereal for breakfast.' },
    { id: 257, en: 'Considerable', cn: '相当大的', status: 'new', example: 'The project requires a considerable amount of time and effort.' },
    { id: 258, en: 'Reverse', cn: '逆转', status: 'new', example: 'We must work quickly to reverse the negative environmental trend.' },
    { id: 259, en: 'Wolfing', cn: '狼吞虎咽', status: 'new', example: 'He was wolfing down his lunch as he was running late.' },
    { id: 260, en: 'Hastily', cn: '急速地，匆忙地', status: 'new', example: 'She hastily grabbed her bag and ran out the door.' },
    { id: 261, en: 'A handful of', cn: '少数，少量', status: 'new', example: 'Only a handful of students passed the extremely difficult test.' },
    { id: 262, en: 'Overlap', cn: '重叠', status: 'new', example: "The two departments' responsibilities sometimes overlap." },
    { id: 263, en: 'Overlapping', cn: '重叠的', status: 'new', example: 'The painter used a technique of overlapping brushstrokes.' },
    { id: 264, en: 'Regardless of', cn: '不管，不顾', status: 'new', example: 'Everyone must follow the rules, regardless of their position.' },
    { id: 265, en: 'emotional intelligence', cn: '情商', status: 'new', example: 'Emotional intelligence is vital for effective leadership.' },
    { id: 266, en: 'Generally', cn: '通常，大体上', status: 'new', example: 'Generally, the weather is quite mild in this region.' },
    { id: 267, en: 'Manipulate', cn: '操纵，控制', status: 'new', example: 'He tried to manipulate the situation to his own advantage.' },
    { id: 268, en: 'Burden', cn: '负荷，重负', status: 'new', example: 'The cost of the repairs became a heavy burden on the family.' },
    { id: 269, en: 'Cope with', cn: '应付，处理', status: 'new', example: 'She is learning how to cope with the stress of her new job.' },
    { id: 270, en: 'Prevailing', cn: '盛行的', status: 'new', example: 'The prevailing opinion is that the market will recover soon.' },
    { id: 271, en: 'Extent', cn: '程度，范围', status: 'new', example: 'The full extent of the damage is still unknown.' },
    { id: 272, en: 'Disgust', cn: '厌恶，反感', status: 'new', example: 'His rude behavior filled the other guests with disgust.' },
    { id: 273, en: 'Perceptive', cn: '有洞察力的', status: 'new', example: 'She is a highly perceptive analyst of social trends.' },
    { id: 274, en: 'Assume', cn: '假设，假定', status: 'new', example: 'I assume you will be arriving around noon for the meeting.' },
    { id: 275, en: 'Inevitable', cn: '不可避免的', status: 'new', example: 'Change is an inevitable part of life and growth.' },
    { id: 276, en: 'Transfer', cn: '转移', status: 'new', example: 'We need to transfer the files from the old server to the new one.' },
    { id: 277, en: 'Emergency', cn: '紧急情况', status: 'new', example: 'In case of an emergency, call this number immediately.' },
    { id: 278, en: 'Prescription', cn: '处方，药方', status: 'new', example: 'The doctor wrote a prescription for a new medication.' },
    { id: 279, en: 'Residence', cn: '住所', status: 'new', example: 'The official residence of the Prime Minister is in the capital.' },
    { id: 280, en: 'Truck', cn: '卡车', status: 'new', example: 'A large delivery truck blocked the narrow street.' },
    { id: 281, en: 'Navigate', cn: '导航，浏览', status: 'new', example: 'The captain used the stars to navigate across the ocean.' },
    { id: 282, en: 'Navigation', cn: '导航', status: 'new', example: 'Modern navigation systems use GPS satellites.' },
    { id: 283, en: 'Heatedly', cn: '激昂地', status: 'new', example: 'They argued heatedly about the political situation.' },
    { id: 284, en: 'cultural identity', cn: '文化认同', status: 'new', example: 'Language plays a critical role in forming cultural identity.' },
    { id: 285, en: 'Twofold', cn: '双重的', status: 'new', example: 'The problem is twofold: financial constraints and a lack of staff.' },
    { id: 286, en: 'garbage sorting and recycling', cn: '垃圾分类和回收', status: 'new', example: 'The city implemented strict rules for garbage sorting and recycling.' },
    { id: 287, en: 'Competence', cn: '能力，胜任', status: 'new', example: 'Her technical competence is respected throughout the industry.' },
    { id: 288, en: 'Proficiency', cn: '熟练', status: 'new', example: 'He demonstrated a high proficiency in three different languages.' },
    { id: 289, en: 'Foster', cn: '促进，培养', status: 'new', example: 'The program aims to foster a sense of community among residents.' },
    { id: 290, en: 'Appropriate', cn: '合适的', status: 'new', example: 'Please wear appropriate attire for the formal ceremony.' },
    { id: 291, en: 'Advent', cn: '出现，到来', status: 'new', example: 'The advent of the internet changed everything about media consumption.' },
    { id: 292, en: 'Rationally', cn: '理性地', status: 'new', example: 'We need to approach this problem calmly and rationally.' },
    { id: 293, en: 'Substitute', cn: '替代品', status: 'new', example: 'They used margarine as a substitute for butter in the recipe.' },
    { id: 294, en: 'Servant', cn: '仆人', status: 'new', example: 'The wealthy family employed several household servants.' },
    { id: 295, en: 'Hastily', cn: '匆忙地', status: 'new', example: 'The decision was made too hastily and without proper review.' },
    { id: 296, en: 'Rushed', cn: '草率的', status: 'new', example: 'The report looked rushed and contained many errors.' },
    { id: 297, en: 'Script', cn: '剧本，讲稿', status: 'new', example: 'The director is currently reviewing the final script for the film.' },
    { id: 298, en: 'Courtesy', cn: '礼貌，彬彬有礼', status: 'new', example: 'Always treat your customers with courtesy and respect.' },
    { id: 299, en: 'Sheet', cn: '床单，纸张', status: 'new', example: 'Please print the results on a separate sheet of paper.' },
    { id: 300, en: 'Roughly', cn: '粗略地，大约', status: 'new', example: 'The estimated cost is roughly five thousand dollars.' },
    { id: 301, en: 'Matter', cn: '事项，事件', status: 'new', example: 'This is a serious matter that requires immediate attention.' },
    { id: 302, en: 'Process', cn: '过程，处理', status: 'new', example: 'We are currently in the process of interviewing new candidates.' },
    { id: 303, en: 'Casual', cn: '随便的，非正式的', status: 'new', example: 'The office dress code is casual on Fridays.' },
    { id: 304, en: 'Universal', cn: '普遍的', status: 'new', example: 'The need for clean water is a universal human right.' },
    { id: 305, en: 'Contraction', cn: '收缩，缩小', status: 'new', example: 'Muscle contraction is necessary for all physical movement.' },
    { id: 306, en: 'Abstract', cn: '抽象的', status: 'new', example: 'Her painting style is quite abstract and difficult to define.' },
    { id: 307, en: 'Rural', cn: '乡村的', status: 'new', example: 'She prefers the quiet life of the rural countryside.' },
    { id: 308, en: 'Playfully', cn: '开玩笑地', status: 'new', example: 'The dog playfully tugged at the end of the rope.' },
    { id: 309, en: 'Ironically', cn: '具有讽刺意味的是', status: 'new', example: 'Ironically, the fire station burned down last night.' },
    { id: 310, en: 'Variation', cn: '变化，变动', status: 'new', example: 'There is a wide variation in prices among different stores.' },
    { id: 311, en: 'Deliberately', cn: '故意地', status: 'new', example: 'He deliberately ignored the instruction to save time.' },
    { id: 312, en: 'Vibe', cn: '氛围', status: 'new', example: 'The cafe has a very relaxed and positive vibe.' },
    { id: 313, en: 'Tragedy', cn: '悲剧', status: 'new', example: 'The plane crash was an unimaginable tragedy.' },
    { id: 314, en: 'Reserve', cn: '预定，保留', status: 'new', example: 'I need to reserve a table at the restaurant for Saturday evening.' },
    { id: 315, en: 'Prominent', cn: '重要的，著名的', status: 'new', example: 'She is a prominent figure in the field of education.' },
    { id: 316, en: 'Compelling', cn: '令人信服的', status: 'new', example: 'His argument was so compelling that the audience immediately agreed.' },
    { id: 317, en: 'Inverse', cn: '相反的，倒转的', status: 'new', example: 'There is an inverse relationship between exercise and weight gain.' },
    { id: 318, en: 'Counter', cn: '相反的', status: 'new', example: 'We need to propose a counter argument to their proposal.' },
    { id: 319, en: 'Reverse', cn: '逆转，相反', status: 'new', example: 'She put the car in reverse to back out of the parking space.' },
    { id: 320, en: 'Opposite', cn: '相反的', status: 'new', example: 'Black is the opposite of white.' },
    { id: 321, en: 'To the contrary', cn: '与此相反', status: 'new', example: 'Unless you hear something to the contrary, assume the meeting is still on.' },
    { id: 322, en: 'Replicate', cn: '重复，复制', status: 'new', example: 'Scientists are struggling to replicate the initial experimental results.' },
    { id: 323, en: 'Distinctive', cn: '独特的', status: 'new', example: 'The building has a distinctive architectural style.' },
    { id: 324, en: 'Critique', cn: '评论，评判', status: 'new', example: 'The author wrote a harsh critique of the new novel.' },
    { id: 325, en: 'Trump', cn: '胜过', status: 'new', example: 'Honesty should always trump any short-term personal gain.' },
    { id: 326, en: 'Profound', cn: '深刻的', status: 'new', example: 'The experience left a profound impact on her view of the world.' },
    { id: 327, en: 'touch on', cn: '触及，提及', status: 'new', example: 'The speech will touch on the main points of the new financial policy.' },
    { id: 328, en: 'Intuitive', cn: '直觉的', status: 'new', example: 'She has an intuitive grasp of complex social situations.' },
    { id: 329, en: 'Technical', cn: '技术的，专业的', status: 'new', example: 'The software requires a high level of technical expertise.' },
    { id: 330, en: 'Frustration', cn: '懊恼，沮丧', status: 'new', example: 'He vented his frustration by slamming the door shut.' },
    { id: 331, en: 'Implement', cn: '执行，贯彻', status: 'new', example: 'We must implement the safety procedures immediately.' },
    { id: 332, en: 'Vision', cn: '远见，构想', status: 'new', example: "The founder had a clear vision for the company's future growth." },
    { id: 333, en: 'Involve', cn: '涉及，包含', status: 'new', example: 'The renovation project will involve changing the entire layout of the kitchen.' },
    { id: 334, en: 'Access', cn: '访问，进入', status: 'new', example: 'Only authorized personnel have access to the server room.' },
    { id: 335, en: 'Premium', cn: '高品质的', status: 'new', example: 'They charge a premium price for their handcrafted leather goods.' },
    { id: 336, en: 'Execute', cn: '执行，实施', status: 'new', example: 'The programmer was able to execute the complex code flawlessly.' },
    { id: 337, en: 'Sincere', cn: '真诚的，诚挚的', status: 'new', example: 'Please accept my sincere apologies for the mistake.' },
    { id: 338, en: 'Assess', cn: '评估', status: 'new', example: 'The damage to the building was quickly assessed by an expert.' },
    { id: 339, en: 'a couple of months', cn: '几个月', status: 'new', example: 'The construction project is expected to take a couple of months.' },
    { id: 340, en: 'take credit', cn: '居功', status: 'new', example: 'It is wrong to take credit for the hard work of others.' },
    { id: 341, en: 'Sustained', cn: '持续的，持久的', status: 'new', example: 'The market saw a period of sustained growth over five years.' },
    { id: 342, en: 'Fundamentally', cn: '根本地', status: 'new', example: 'Fundamentally, all humans share the same basic needs.' },
    { id: 343, en: 'Proponent', cn: '支持者，建议者', status: 'new', example: 'She is a strong proponent of renewable energy sources.' },
    { id: 344, en: 'Center', cn: '中心', status: 'new', example: 'The community center is located right in the center of the town.' },
    { id: 345, en: 'Central', cn: '中心的', status: 'new', example: 'The central theme of the novel is the struggle for freedom.' },
  ];

  const [cards, setCards] = useState(() => {
    // 检查本地存储，如果为空，则使用导入的词汇列表
    const saved = localStorage.getItem('focusflow-cards');
    return saved ? JSON.parse(saved) : initialCards;
  });

  const [mode, setMode] = useState('home'); // 'home', 'add', 'quiz', 'result'
  const [quizQueue, setQuizQueue] = useState([]);
  const [currentCardIndex, setCurrentCardIndex] = useState(0);
  const [userAnswer, setUserAnswer] = useState('');
  const [feedback, setFeedback] = useState(null); // 'correct', 'incorrect', or null
  const [stats, setStats] = useState({ correct: 0, wrong: 0 });

  // Input states
  const [newEn, setNewEn] = useState('');
  const [newCn, setNewCn] = useState('');
  const [newExample, setNewExample] = useState('');

  // API loading states
  const [isExampleLoading, setIsExampleLoading] = useState(false);
  const [isTtsLoading, setIsTtsLoading] = useState(false);

  // Persist to local storage
  useEffect(() => {
    localStorage.setItem('focusflow-cards', JSON.stringify(cards));
  }, [cards]);
  
  // Audio Ref
  const audioRef = useRef(null);

  // --- Gemini Feature 1: Example Sentence Generation ---
  const handleGenerateExample = async () => {
    if (!newEn.trim()) return;
    setIsExampleLoading(true);
    setNewExample('');
    try {
        const prompt = `Generate a single, natural English example sentence for the word: ${newEn.trim()}`;
        const exampleText = await callGeminiApi(prompt);
        setNewExample(exampleText.replace(/\n/g, ' ').trim()); // Clean up
    } catch (e) {
        console.error("Error generating example:", e);
        setNewExample("无法生成例句，请检查网络或重试。");
    } finally {
        setIsExampleLoading(false);
    }
  };

  // --- Gemini Feature 2: Text-to-Speech Pronunciation ---
  const handleTts = async (word) => {
      if (isTtsLoading) return;
      setIsTtsLoading(true);
      
      try {
          const audioUrl = await callTtsApi(word);
          if (audioRef.current) {
              // Clean up previous URL object to prevent memory leak
              if (audioRef.current.src) {
                  URL.revokeObjectURL(audioRef.current.src);
              }
              audioRef.current.src = audioUrl;
              await audioRef.current.play();
          }
      } catch (e) {
          console.error("Error playing audio:", e);
      } finally {
          setIsTtsLoading(false);
      }
  };

  // --- Logic ---

  const addCard = () => {
    if (!newEn.trim() || !newCn.trim()) return;
    const newCard = {
      id: Date.now(),
      en: newEn.trim(),
      cn: newCn.trim(),
      example: newExample.trim(), // Include generated example
      status: 'new'
    };
    setCards([...cards, newCard]);
    setNewEn('');
    setNewCn('');
    setNewExample('');
  };

  const deleteCard = (id) => {
    setCards(cards.filter(c => c.id !== id));
  };

  const startQuiz = () => {
    if (cards.length === 0) return;
    // Shuffle cards for the queue
    const shuffled = [...cards].sort(() => Math.random() - 0.5);
    setQuizQueue(shuffled);
    setCurrentCardIndex(0);
    setStats({ correct: 0, wrong: 0 });
    setFeedback(null);
    setUserAnswer('');
    setMode('quiz');
  };

  const handleAnswerSubmit = (e) => {
    e.preventDefault();
    if (feedback) return; // Prevent double submission

    const currentCard = quizQueue[currentCardIndex];
    // Case-insensitive, space-trimmed comparison for Chinese meaning
    const isCorrect = userAnswer.trim() === currentCard.cn.trim(); 

    if (isCorrect) {
      setFeedback('correct');
      setStats(prev => ({ ...prev, correct: prev.correct + 1 }));
      // Wait a bit then move to next
      setTimeout(() => {
        moveToNextCard(true);
      }, 1000);
    } else {
      setFeedback('incorrect');
      setStats(prev => ({ ...prev, wrong: prev.wrong + 1 }));
    }
  };

  const handleIncorrectNext = () => {
    // Logic: Move current card to the end of the queue to review again later
    const currentCard = quizQueue[currentCardIndex];
    const newQueue = [...quizQueue];
    
    // Append the card to the end of the queue
    newQueue.push(currentCard);
    
    setQuizQueue(newQueue);
    moveToNextCard(false);
  };

  const moveToNextCard = (wasCorrect) => {
    setFeedback(null);
    setUserAnswer('');
    
    if (currentCardIndex + 1 < quizQueue.length) {
      setCurrentCardIndex(prev => prev + 1);
    } else {
      // End of session
      setMode('result');
    }
  };

  // --- Components ---

  const Header = ({ title, subtitle }) => (
    <div className="mb-8 text-center animate-fade-in-down">
      <h1 className="text-3xl font-light tracking-tight text-slate-800">{title}</h1>
      {subtitle && <p className="text-slate-500 mt-2 text-sm">{subtitle}</p>}
    </div>
  );

  const currentQuizCard = quizQueue[currentCardIndex];

  return (
    <div className="min-h-screen bg-[#Fdfdfd] text-slate-800 font-sans selection:bg-indigo-100 flex flex-col items-center py-12 px-4 sm:px-6">
      
      <audio ref={audioRef} />

      {/* Decorative Background Blobs */}
      <div className="fixed top-0 left-0 w-64 h-64 bg-indigo-50 rounded-full blur-3xl -z-10 opacity-60 mix-blend-multiply filter animate-blob"></div>
      <div className="fixed top-0 right-0 w-64 h-64 bg-rose-50 rounded-full blur-3xl -z-10 opacity-60 mix-blend-multiply filter animate-blob animation-delay-2000"></div>
      <div className="fixed -bottom-8 left-20 w-72 h-72 bg-slate-100 rounded-full blur-3xl -z-10 opacity-60 mix-blend-multiply filter animate-blob animation-delay-4000"></div>

      <div className="w-full max-w-md bg-white/80 backdrop-blur-xl rounded-3xl shadow-[0_8px_30px_rgb(0,0,0,0.04)] border border-white/50 p-6 sm:p-8 min-h-[600px] relative overflow-hidden transition-all duration-500">
        
        {/* Navigation / Top Bar */}
        {mode !== 'home' && (
          <button 
            onClick={() => {
              setMode('home');
              setFeedback(null);
              // Clean up audio object URL if present
              if (audioRef.current?.src) URL.revokeObjectURL(audioRef.current.src);
            }}
            className="absolute top-6 left-6 p-2 rounded-full hover:bg-slate-100 text-slate-400 transition-colors"
          >
            <RotateCcw size={20} />
          </button>
        )}

        {/* --- HOME MODE --- */}
        {mode === 'home' && (
          <div className="h-full flex flex-col">
            <Header title="FocusFlow" subtitle="极简记忆空间" />
            
            <div className="flex-1 flex flex-col justify-center space-y-4">
              <div className="text-center mb-8">
                <div className="text-6xl font-extralight text-slate-900 mb-2">{cards.length}</div>
                <div className="text-xs font-medium uppercase tracking-widest text-slate-400">当前词汇量</div>
              </div>

              <button 
                onClick={startQuiz}
                disabled={cards.length === 0}
                className="group relative w-full py-4 bg-slate-900 text-white rounded-2xl shadow-lg shadow-slate-200 hover:shadow-xl hover:-translate-y-0.5 transition-all disabled:opacity-50 disabled:cursor-not-allowed flex items-center justify-center space-x-2 overflow-hidden"
              >
                <div className="absolute inset-0 w-full h-full bg-gradient-to-r from-indigo-500 to-indigo-600 opacity-0 group-hover:opacity-100 transition-opacity duration-500"></div>
                <Brain size={20} className="relative z-10" />
                <span className="relative z-10 font-medium">开始复习</span>
              </button>

              <button 
                onClick={() => setMode('add')}
                className="w-full py-4 bg-white border border-slate-200 text-slate-700 rounded-2xl hover:bg-slate-50 hover:border-slate-300 transition-all flex items-center justify-center space-x-2"
              >
                <Plus size={20} />
                <span className="font-medium">录入新词</span>
              </button>
            </div>
          </div>
        )}

        {/* --- ADD MODE --- */}
        {mode === 'add' && (
          <div className="h-full flex flex-col">
            <Header title="录入词汇" />
            
            <div className="space-y-4 mb-8">
              <div className="space-y-1">
                <label className="text-xs font-bold text-slate-400 uppercase tracking-wider ml-1">英文单词</label>
                <input 
                  value={newEn}
                  onChange={(e) => {
                    setNewEn(e.target.value);
                    setNewExample(''); // Clear example on word change
                  }}
                  placeholder="Ex: Ephemeral"
                  className="w-full p-4 bg-slate-50 border-none rounded-2xl text-lg font-medium text-slate-800 placeholder:text-slate-300 focus:ring-2 focus:ring-indigo-100 focus:bg-white transition-all outline-none"
                />
              </div>
              <div className="space-y-1">
                <label className="text-xs font-bold text-slate-400 uppercase tracking-wider ml-1">中文释义</label>
                <input 
                  value={newCn}
                  onChange={(e) => setNewCn(e.target.value)}
                  placeholder="Ex: 短暂的"
                  className="w-full p-4 bg-slate-50 border-none rounded-2xl text-lg font-medium text-slate-800 placeholder:text-slate-300 focus:ring-2 focus:ring-indigo-100 focus:bg-white transition-all outline-none"
                  onKeyDown={(e) => e.key === 'Enter' && addCard()}
                />
              </div>
              
              <div className="space-y-1">
                <div className='flex items-center justify-between'>
                    <label className="text-xs font-bold text-slate-400 uppercase tracking-wider ml-1">例句 (可选)</label>
                    <button 
                        onClick={handleGenerateExample}
                        disabled={!newEn.trim() || isExampleLoading}
                        className="flex items-center space-x-1 text-xs text-indigo-600 font-semibold bg-indigo-50 px-2 py-1 rounded-full hover:bg-indigo-100 transition-colors disabled:opacity-50"
                    >
                        {isExampleLoading ? (
                            <svg className="animate-spin h-3 w-3 text-indigo-500" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24"><circle className="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" strokeWidth="4"></circle><path className="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path></svg>
                        ) : (
                            <Sparkles size={12} />
                        )}
                        <span>{isExampleLoading ? '生成中...' : '生成示例句'}</span>
                    </button>
                </div>
                <textarea 
                  value={newExample}
                  onChange={(e) => setNewExample(e.target.value)}
                  placeholder="或手动输入例句..."
                  rows={2}
                  className="w-full p-4 bg-slate-50 border-none rounded-2xl text-base text-slate-800 placeholder:text-slate-300 focus:ring-2 focus:ring-indigo-100 focus:bg-white transition-all outline-none resize-none"
                />
              </div>

              <button 
                onClick={addCard}
                disabled={!newEn || !newCn}
                className="w-full py-3 bg-slate-900 text-white rounded-xl shadow-md hover:bg-slate-800 transition-all disabled:opacity-50"
              >
                添加
              </button>
            </div>

            <div className="flex-1 overflow-y-auto pr-1 -mr-1">
              <h3 className="text-sm font-semibold text-slate-400 mb-3 ml-1">最近添加</h3>
              <div className="space-y-2">
                {[...cards].reverse().slice(0, 10).map(card => (
                  <div key={card.id} className="group flex flex-col p-3 rounded-xl bg-white border border-slate-100 hover:border-slate-200 transition-all">
                    <div className='flex items-center justify-between'>
                        <div className="font-medium text-slate-800">{card.en}</div>
                        <button 
                            onClick={() => deleteCard(card.id)}
                            className="p-1 text-slate-300 hover:text-rose-400 opacity-0 group-hover:opacity-100 transition-all"
                        >
                            <Trash2 size={16} />
                        </button>
                    </div>
                    <div className="text-sm text-slate-500">{card.cn}</div>
                    {card.example && (
                        <div className="text-xs italic text-slate-400 mt-1 pt-1 border-t border-slate-50/50">{card.example}</div>
                    )}
                  </div>
                ))}
                {cards.length === 0 && (
                  <div className="text-center py-8 text-slate-300 text-sm">
                    暂无单词，开始添加吧
                  </div>
                )}
              </div>
            </div>
          </div>
        )}

        {/* --- QUIZ MODE --- */}
        {mode === 'quiz' && currentQuizCard && (
          <div className="h-full flex flex-col justify-between">
            {/* Progress Bar */}
            <div className="w-full h-1.5 bg-slate-100 rounded-full mb-8 overflow-hidden">
              <div 
                className="h-full bg-indigo-500 transition-all duration-500 ease-out"
                style={{ width: `${((currentCardIndex) / quizQueue.length) * 100}%` }}
              ></div>
            </div>

            <div className="flex-1 flex flex-col items-center justify-center -mt-8">
              <div className="text-xs font-bold text-slate-400 uppercase tracking-widest mb-4">
                Translate to Chinese
              </div>
              
              <div className="flex items-center justify-center space-x-3 mb-12 animate-fade-in-up">
                <div className="text-4xl sm:text-5xl font-semibold text-slate-800 text-center">
                    {currentQuizCard.en}
                </div>
                <button 
                    onClick={() => handleTts(currentQuizCard.en)}
                    disabled={isTtsLoading}
                    className="text-indigo-500 hover:text-indigo-700 disabled:opacity-50 disabled:cursor-wait transition-colors p-2 rounded-full hover:bg-indigo-50"
                    aria-label="Play pronunciation"
                >
                    {isTtsLoading ? (
                        <svg className="animate-spin h-6 w-6 text-indigo-500" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24"><circle className="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" strokeWidth="4"></circle><path className="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path></svg>
                    ) : (
                        <Volume2 size={28} />
                    )}
                </button>
              </div>

              <form onSubmit={handleAnswerSubmit} className="w-full relative group">
                <input 
                  autoFocus
                  value={userAnswer}
                  onChange={(e) => setUserAnswer(e.target.value)}
                  disabled={feedback !== null}
                  placeholder="输入中文含义..."
                  className={`w-full p-5 text-center bg-white border-2 rounded-2xl text-xl text-slate-800 placeholder:text-slate-300 focus:outline-none transition-all shadow-sm
                    ${feedback === 'correct' ? 'border-emerald-400 bg-emerald-50 text-emerald-800' : 
                      feedback === 'incorrect' ? 'border-rose-400 bg-rose-50 text-rose-800' : 
                      'border-slate-100 focus:border-indigo-300 focus:shadow-indigo-100/50'}`}
                />
                
                {/* Visual Feedback Icons */}
                <div className="absolute right-4 top-1/2 -translate-y-1/2 transition-all">
                  {feedback === 'correct' && <Check className="text-emerald-500 animate-scale-in" size={24} />}
                  {feedback === 'incorrect' && <X className="text-rose-500 animate-scale-in" size={24} />}
                </div>
              </form>

              {/* Feedback Area */}
              <div className={`mt-6 w-full text-center transition-all duration-300 ${feedback ? 'opacity-100 translate-y-0' : 'opacity-0 translate-y-2'}`}>
                {feedback === 'correct' && (
                  <div className="text-emerald-600 font-medium flex flex-col items-center">
                    <span>🎉 正确! 这种感觉太棒了。</span>
                    {currentQuizCard.example && (
                        <p className="text-sm italic text-emerald-700/80 mt-2">"{currentQuizCard.example}"</p>
                    )}
                  </div>
                )}

                {feedback === 'incorrect' && (
                  <div className="bg-white p-4 rounded-2xl border border-rose-100 shadow-sm w-full animate-shake">
                    <div className="text-xs text-rose-400 font-bold uppercase mb-1">正确答案</div>
                    <div className="text-lg text-slate-800 font-medium mb-4">{currentQuizCard.cn}</div>
                    {currentQuizCard.example && (
                        <div className="text-sm text-slate-600 mb-4 bg-slate-50 p-3 rounded-lg border border-slate-100">
                            <span className="text-xs font-semibold text-slate-400 block mb-1">例句参考:</span>
                            <span className='italic'>"{currentQuizCard.example}"</span>
                        </div>
                    )}
                    <button 
                      onClick={handleIncorrectNext}
                      className="w-full py-2.5 bg-slate-900 text-white text-sm rounded-xl hover:bg-slate-800 transition-colors flex items-center justify-center space-x-2"
                    >
                      <span>记住了，放入队尾</span>
                      <ChevronRight size={16} />
                    </button>
                    <div className="text-xs text-slate-400 mt-2">该单词稍后会再次出现</div>
                  </div>
                )}
              </div>
            </div>

            <div className="text-center text-xs text-slate-300 font-medium mt-4">
               {quizQueue.length - currentCardIndex} words remaining in queue
            </div>
          </div>
        )}

        {/* --- RESULT MODE --- */}
        {mode === 'result' && (
          <div className="h-full flex flex-col items-center justify-center text-center">
             <div className="w-20 h-20 bg-emerald-100 rounded-full flex items-center justify-center mb-6 text-emerald-600 animate-scale-in">
               <Check size={40} strokeWidth={3} />
             </div>
             
             <h2 className="text-2xl font-semibold text-slate-800 mb-2">复习完成!</h2>
             <p className="text-slate-500 mb-8 max-w-[200px]">你已经完成了当前队列中的所有单词复习。</p>

             <div className="grid grid-cols-2 gap-4 w-full mb-8">
               <div className="bg-slate-50 p-4 rounded-2xl">
                 <div className="text-2xl font-bold text-slate-800">{stats.correct}</div>
                 <div className="text-xs text-slate-400 uppercase font-bold">正确</div>
               </div>
               <div className="bg-slate-50 p-4 rounded-2xl">
                 <div className="text-2xl font-bold text-rose-500">{stats.wrong}</div>
                 <div className="text-xs text-rose-300 uppercase font-bold">错误次数</div>
               </div>
             </div>

             <button 
                onClick={() => setMode('home')}
                className="w-full py-3 bg-slate-900 text-white rounded-xl shadow-lg hover:bg-slate-800 transition-all"
              >
                返回主页
              </button>
          </div>
        )}

      </div>

      <style>{`
        @keyframes blob {
          0% { transform: translate(0px, 0px) scale(1); }
          33% { transform: translate(30px, -50px) scale(1.1); }
          66% { transform: translate(-20px, 20px) scale(0.9); }
          100% { transform: translate(0px, 0px) scale(1); }
        }
        .animate-blob {
          animation: blob 7s infinite;
        }
        .animation-delay-2000 {
          animation-delay: 2s;
        }
        .animation-delay-4000 {
          animation-delay: 4s;
        }
        .animate-scale-in {
          animation: scaleIn 0.3s cubic-bezier(0.175, 0.885, 0.32, 1.275);
        }
        @keyframes scaleIn {
          from { transform: scale(0); opacity: 0; }
          to { transform: scale(1); opacity: 1; }
        }
        .animate-fade-in-up {
          animation: fadeInUp 0.5s ease-out;
        }
        @keyframes fadeInUp {
          from { opacity: 0; transform: translateY(10px); }
          to { opacity: 1; transform: translateY(0); }
        }
        .animate-shake {
          animation: shake 0.5s cubic-bezier(.36,.07,.19,.97) both;
        }
        @keyframes shake {
          10%, 90% { transform: translate3d(-1px, 0, 0); }
          20%, 80% { transform: translate3d(2px, 0, 0); }
          30%, 50%, 70% { transform: translate3d(-4px, 0, 0); }
          40%, 60% { transform: translate3d(4px, 0, 0); }
        }
      `}</style>
    </div>
  );
};

export default FocusFlow;
