new file mode 100644
@@ -0,0 +1,125 @@

import { streamText } from "ai";

import { myProvider } from "@/lib/ai/providers";

import { auth } from "@/app/(auth)/auth";

import { ChatSDKError } from "@/lib/errors";

 

export const maxDuration = 60;

 

export async function POST(request: Request) {

  try {

    const session = await auth();

 

    if (!session?.user) {

      return new ChatSDKError("unauthorized:chat").toResponse();

    }

 

    const { sourceText } = await request.json();

 

    if (!sourceText || sourceText.trim().length < 50) {

      return Response.json(

        { error: "O texto deve ter pelo menos 50 caracteres." },

        { status: 400 }

      );

    }

 

    const prompt = `Você é um gerador de perguntas de quiz. Baseado no texto fornecido abaixo, crie exatamente 10 perguntas de múltipla escolha com 4 opções cada.

 

TEXTO:

${sourceText}

 

INSTRUÇÕES:

- Gere exatamente 10 perguntas baseadas no conteúdo do texto

- Cada pergunta deve ter 4 opções de resposta (A, B, C, D)

- Apenas uma opção deve estar correta

- Inclua uma explicação detalhada para cada resposta correta

- As perguntas devem cobrir diferentes aspectos do texto

- Retorne APENAS um objeto JSON válido, sem texto adicional

 

FORMATO DE RESPOSTA (JSON):

{

  "questions": [

    {

      "question": "texto da pergunta",

      "options": ["opção A", "opção B", "opção C", "opção D"],

      "correctAnswer": 0,

      "explanation": "explicação detalhada da resposta correta"

    }

  ]

}

 

O campo "correctAnswer" deve ser o índice da opção correta (0, 1, 2 ou 3).

Gere exatamente 10 perguntas.`;

 

    const result = await streamText({

      model: myProvider.languageModel("chat-model-extended"),

      prompt: prompt,

      temperature: 0.7,

    });

 

    let fullText = "";

    for await (const textPart of result.textStream) {

      fullText += textPart;

    }

 

    // Try to extract JSON from the response

    let jsonResponse;

    try {

      // Remove markdown code blocks if present

      const cleanedText = fullText

        .replace(/```json\n?/g, "")

        .replace(/```\n?/g, "")

        .trim();

 

      jsonResponse = JSON.parse(cleanedText);

    } catch (parseError) {

      console.error("Failed to parse AI response:", fullText);

      return Response.json(

        { error: "Erro ao processar resposta da IA" },

        { status: 500 }

      );

    }

 

    // Validate the response structure

    if (

      !jsonResponse.questions ||

      !Array.isArray(jsonResponse.questions) ||

      jsonResponse.questions.length !== 10

    ) {

      return Response.json(

        { error: "Formato de resposta inválido da IA" },

        { status: 500 }

      );

    }

 

    // Validate each question

    for (const q of jsonResponse.questions) {

      if (

        !q.question ||

        !Array.isArray(q.options) ||

        q.options.length !== 4 ||

        typeof q.correctAnswer !== "number" ||

        q.correctAnswer < 0 ||

        q.correctAnswer > 3 ||

        !q.explanation

      ) {

        return Response.json(

          { error: "Formato de pergunta inválido" },

          { status: 500 }

        );

      }

    }

 

    return Response.json(jsonResponse);

  } catch (error) {

    console.error("Error generating quiz:", error);

 

    if (error instanceof ChatSDKError) {

      return error.toResponse();

    }

 

    return Response.json(

      { error: "Erro ao gerar quiz. Por favor, tente novamente." },

      { status: 500 }

    );

  }

}
Large diff (317 lines). Showing first 300 lines.
new file mode 100644
@@ -0,0 +1,315 @@

'use client';

 

import { useState } from 'react';

 

interface Question {

  question: string;

  options: string[];

  correctAnswer: number;

  explanation: string;

}

 

export default function QuizPage() {

  const [sourceText, setSourceText] = useState('');

  const [questions, setQuestions] = useState<Question[]>([]);

  const [currentQuestionIndex, setCurrentQuestionIndex] = useState(0);

  const [selectedAnswers, setSelectedAnswers] = useState<number[]>([]);

  const [showResults, setShowResults] = useState(false);

  const [isGenerating, setIsGenerating] = useState(false);

  const [quizStarted, setQuizStarted] = useState(false);

 

  const generateQuestions = async () => {

    if (!sourceText.trim()) {

      alert('Por favor, insira um texto para gerar o quiz.');

      return;

    }

 

    if (sourceText.trim().length < 50) {

      alert('O texto deve ter pelo menos 50 caracteres.');

      return;

    }

 

    setIsGenerating(true);

 

    try {

      const response = await fetch('/api/quiz/generate', {

        method: 'POST',

        headers: {

          'Content-Type': 'application/json',

        },

        body: JSON.stringify({ sourceText }),

      });

 

      if (!response.ok) {

        const error = await response.json();

        throw new Error(error.error || 'Erro ao gerar quiz');

      }

 

      const data = await response.json();

 

      // Randomize questions

      const shuffled = [...data.questions].sort(() => Math.random() - 0.5);

      setQuestions(shuffled);

      setSelectedAnswers(new Array(10).fill(-1));

      setQuizStarted(true);

    } catch (error) {

      console.error('Error generating quiz:', error);

      alert(error instanceof Error ? error.message : 'Erro ao gerar quiz. Por favor, tente novamente.');

    } finally {

      setIsGenerating(false);

    }

  };

 

  const handleAnswerSelect = (answerIndex: number) => {

    const newAnswers = [...selectedAnswers];

    newAnswers[currentQuestionIndex] = answerIndex;

    setSelectedAnswers(newAnswers);

  };

 

  const handleNext = () => {

    if (currentQuestionIndex < questions.length - 1) {

      setCurrentQuestionIndex(currentQuestionIndex + 1);

    }

  };

 

  const handlePrevious = () => {

    if (currentQuestionIndex > 0) {

      setCurrentQuestionIndex(currentQuestionIndex - 1);

    }

  };

 

  const handleSubmit = () => {

    if (selectedAnswers.includes(-1)) {

      alert('Por favor, responda todas as perguntas antes de finalizar.');

      return;

    }

    setShowResults(true);

  };

 

  const calculateScore = () => {

    let correct = 0;

    questions.forEach((q, index) => {

      if (selectedAnswers[index] === q.correctAnswer) {

        correct++;

      }

    });

    return correct;

  };

 

  const restartQuiz = () => {

    setSourceText('');

    setQuestions([]);

    setCurrentQuestionIndex(0);

    setSelectedAnswers([]);

    setShowResults(false);

    setQuizStarted(false);

  };

 

  if (!quizStarted) {

    return (

      <div className="min-h-screen bg-gradient-to-br from-blue-50 to-indigo-100 py-12 px-4">

        <div className="max-w-4xl mx-auto">

          <div className="bg-white rounded-lg shadow-xl p-8">

            <h1 className="text-4xl font-bold text-center text-indigo-900 mb-2">

              Gerador de Quiz

            </h1>

            <p className="text-center text-gray-600 mb-8">

              Cole um texto e gere 10 perguntas aleatórias com correção automática

            </p>

 

            <div className="mb-6">

              <label className="block text-sm font-medium text-gray-700 mb-2">

                Cole ou digite o texto base para o quiz:

              </label>

              <textarea

                className="w-full h-64 p-4 border border-gray-300 rounded-lg focus:ring-2 focus:ring-indigo-500 focus:border-transparent resize-none"

                placeholder="Cole seu texto aqui..."

                value={sourceText}

                onChange={(e) => setSourceText(e.target.value)}

              />

            </div>

 

            <button

              onClick={generateQuestions}

              disabled={isGenerating || !sourceText.trim()}

              className="w-full bg-indigo-600 text-white py-3 px-6 rounded-lg font-semibold hover:bg-indigo-700 disabled:bg-gray-400 disabled:cursor-not-allowed transition-colors"

            >

              {isGenerating ? 'Gerando Quiz...' : 'Gerar Quiz'}

            </button>

          </div>

        </div>

      </div>

    );

  }

 

  if (showResults) {

    const score = calculateScore();

 

    return (

      <div className="min-h-screen bg-gradient-to-br from-blue-50 to-indigo-100 py-12 px-4">

        <div className="max-w-4xl mx-auto">

          <div className="bg-white rounded-lg shadow-xl p-8">

            <h1 className="text-4xl font-bold text-center text-indigo-900 mb-6">

              Resultado do Quiz

            </h1>

 

            <div className="text-center mb-8">

              <div className="inline-block bg-indigo-100 rounded-full px-8 py-4 mb-4">

                <p className="text-6xl font-bold text-indigo-600">{score}</p>

                <p className="text-gray-600">de 10 pontos</p>

              </div>

 

              <p className="text-2xl font-semibold text-gray-700">

                {score >= 9 ? 'Excelente!' : score >= 7 ? 'Muito Bom!' : score >= 5 ? 'Bom!' : 'Continue estudando!'}

              </p>

            </div>

 

            <div className="space-y-6">

              {questions.map((q, index) => {

                const isCorrect = selectedAnswers[index] === q.correctAnswer;

 

                return (

                  <div

                    key={index}

                    className={`p-6 rounded-lg border-2 ${

                      isCorrect ? 'border-green-500 bg-green-50' : 'border-red-500 bg-red-50'

                    }`}

                  >

                    <div className="flex items-start gap-3 mb-3">

                      <span className="flex-shrink-0 w-8 h-8 rounded-full bg-indigo-600 text-white flex items-center justify-center font-bold">

                        {index + 1}

                      </span>

                      <p className="font-semibold text-gray-800 flex-1">{q.question}</p>

                      <span className="text-2xl">

                        {isCorrect ? '✓' : '✗'}

                      </span>

                    </div>

 

                    <div className="ml-11 space-y-2">

                      <p className="text-sm">

                        <span className="font-medium">Sua resposta: </span>

                        <span className={isCorrect ? 'text-green-700' : 'text-red-700'}>

                          {q.options[selectedAnswers[index]]}

                        </span>

                      </p>

 

                      {!isCorrect && (

                        <p className="text-sm">

                          <span className="font-medium">Resposta correta: </span>

                          <span className="text-green-700">

                            {q.options[q.correctAnswer]}

                          </span>

                        </p>

                      )}

 

                      <div className="mt-3 p-3 bg-white rounded border border-gray-200">

                        <p className="text-sm font-medium text-gray-700 mb-1">Explicação:</p>

                        <p className="text-sm text-gray-600">{q.explanation}</p>

                      </div>

                    </div>

                  </div>

                );

              })}

            </div>

 

            <button

              onClick={restartQuiz}

              className="w-full mt-8 bg-indigo-600 text-white py-3 px-6 rounded-lg font-semibold hover:bg-indigo-700 transition-colors"

            >

              Fazer Novo Quiz

            </button>

          </div>

        </div>

      </div>

    );

  }

 

  const currentQuestion = questions[currentQuestionIndex];

 

  return (

    <div className="min-h-screen bg-gradient-to-br from-blue-50 to-indigo-100 py-12 px-4">

      <div className="max-w-4xl mx-auto">

        <div className="bg-white rounded-lg shadow-xl p-8">

          <div className="mb-6">

            <div className="flex justify-between items-center mb-4">

              <h2 className="text-2xl font-bold text-indigo-900">

                Pergunta {currentQuestionIndex + 1} de {questions.length}

              </h2>

              <span className="text-sm text-gray-600">

                {selectedAnswers.filter(a => a !== -1).length} respondidas

              </span>

            </div>

 

            <div className="w-full bg-gray-200 rounded-full h-2">

              <div

                className="bg-indigo-600 h-2 rounded-full transition-all duration-300"

                style={{ width: `${((currentQuestionIndex + 1) / questions.length) * 100}%` }}

              />

            </div>

          </div>

 

          <div className="mb-8">

            <h3 className="text-xl font-semibold text-gray-800 mb-6">

              {currentQuestion.question}

            </h3>

 

            <div className="space-y-3">

              {currentQuestion.options.map((option, index) => (

                <button

                  key={index}

                  onClick={() => handleAnswerSelect(index)}

                  className={`w-full text-left p-4 rounded-lg border-2 transition-all ${

                    selectedAnswers[currentQuestionIndex] === index

                      ? 'border-indigo-600 bg-indigo-50'

                      : 'border-gray-200 hover:border-indigo-300 bg-white'

                  }`}

                >

                  <div className="flex items-center gap-3">

                    <div

                      className={`w-6 h-6 rounded-full border-2 flex items-center justify-center ${

                        selectedAnswers[currentQuestionIndex] === index

                          ? 'border-indigo-600 bg-indigo-600'

                          : 'border-gray-300'

                      }`}

                    >

                      {selectedAnswers[currentQuestionIndex] === index && (

                        <div className="w-3 h-3 bg-white rounded-full" />

                      )}

                    </div>

                    <span className="text-gray-800">{option}</span>

                  </div>

                </button>

              ))}

            </div>

          </div>

 

          <div className="flex justify-between gap-4">

            <button

              onClick={handlePrevious}

              disabled={currentQuestionIndex === 0}

              className="px-6 py-3 rounded-lg font-semibold border-2 border-gray-300 text-gray-700 hover:bg-gray-50 disabled:opacity-50 disabled:cursor-not-allowed transition-colors"

            >

              Anterior

            </button>

 

            {currentQuestionIndex === questions.length - 1 ? (

              <button

                onClick={handleSubmit}

                className="px-6 py-3 rounded-lg font-semibold bg-green-600 text-white hover:bg-green-700 transition-colors"
Showing 300 of 317 lines
Load Next 17 Lines
@@ -114,6 +114,18 @@ export function AppSidebar({ user }: { user: User | undefined }) {

        </SidebarHeader>

        <SidebarContent>

          <SidebarHistory user={user} />

          {user && (

            <div className="px-2 py-2">

              <Link

                href="/quiz"

                onClick={() => setOpenMobile(false)}

                className="flex items-center gap-2 px-3 py-2 rounded-md hover:bg-muted transition-colors text-sm"

              >

                <span>📝</span>

                <span>Quiz Generator</span>

              </Link>

            </div>

          )}

        </SidebarContent>

        <SidebarFooter>{user && <SidebarUserNav user={user} />}</SidebarFooter>

      </Sidebar>
