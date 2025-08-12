```markdown
# Detailed Implementation Plan for Aplikasi Survei Kepuasan Masyarakat Terhubung ke Spreadsheet

## 1. Environment Configuration
- **Create `.env.local` in the project root.**  
  Add the following variables:
  ```env
  GOOGLE_SHEET_ID=14tS57rS5Y5UT170zlmylZ6eDpSNaYCZfW2mrLx7_tbo
  GOOGLE_CLIENT_EMAIL=your-service-account-email@example.iam.gserviceaccount.com
  GOOGLE_PRIVATE_KEY="your-private-key-with-escaped-newlines"
  ```
- **Notes:**  
  • Replace the sample values with your actual Google Cloud service account credentials.  
  • Ensure that any newline characters in `GOOGLE_PRIVATE_KEY` are properly escaped (using `\n`).

## 2. Google Sheets Integration
- **File:** `src/lib/googleSheets.ts`  
- **Changes / Implementation Steps:**
  1. Import the necessary module:
     ```typescript
     import { google } from 'googleapis';
     ```
  2. Instantiate a JWT client using the environment variables:
     ```typescript
     const jwtClient = new google.auth.JWT(
       process.env.GOOGLE_CLIENT_EMAIL,
       undefined,
       process.env.GOOGLE_PRIVATE_KEY?.replace(/\\n/g, '\n'),
       ['https://www.googleapis.com/auth/spreadsheets']
     );
     ```
  3. Create and export an async function `appendRowToSheet` to append a row into the spreadsheet:
     ```typescript
     export async function appendRowToSheet(row: any[]): Promise<any> {
       const sheets = google.sheets({ version: 'v4', auth: jwtClient });
       const response = await sheets.spreadsheets.values.append({
         spreadsheetId: process.env.GOOGLE_SHEET_ID!,
         range: 'Sheet1!A1', // Adjust the sheet name and range if needed
         valueInputOption: 'USER_ENTERED',
         requestBody: { values: [row] }
       });
       return response.data;
     }
     ```
  4. Implement error handling with try-catch as needed in higher-level functions.

## 3. API Endpoint for Survey Submission
- **File:** `src/app/api/survey/route.ts`  
- **Changes / Implementation Steps:**
  1. Create the file if it does not exist.
  2. Import necessary utilities:
     ```typescript
     import { NextResponse } from 'next/server';
     import { appendRowToSheet } from '@/lib/googleSheets';
     ```
  3. Define the POST handler:
     ```typescript
     export async function POST(request: Request) {
       try {
         const body = await request.json();
         // Validate required fields
         if (!body.name || !body.email || !body.rating) {
           return NextResponse.json(
             { error: 'Field "name", "email", and "rating" are required.' },
             { status: 400 }
           );
         }
         // Prepare a row with a timestamp and survey data
         const row = [
           new Date().toISOString(),
           body.name,
           body.email,
           body.rating,
           body.feedback || ''
         ];
         await appendRowToSheet(row);
         return NextResponse.json({ message: 'Survei berhasil dikirim' });
       } catch (error: any) {
         return NextResponse.json(
           { error: error.message || 'Terjadi kesalahan.' },
           { status: 500 }
         );
       }
     }
     ```
  4. Ensure proper JSON request parsing and error handling for robust API behavior.

## 4. Survey Form Page (Client Side UI)
- **File:** `src/app/survey/page.tsx`  
- **Changes / Implementation Steps:**
  1. Mark the file as a client component by including `"use client";` at the top.
  2. Import React hooks and UI components:
     ```typescript
     "use client";
     import { useState } from 'react';
     import { Input } from '@/components/ui/input';
     import { Textarea } from '@/components/ui/textarea';
     import { Button } from '@/components/ui/button';
     ```
  3. Build the survey form with the following fields:
     - **Nama:** Text input for respondent's name.
     - **Email:** Email input field.
     - **Rating Kepuasan:** Input or select/dropdown (e.g., values 1 - 5) representing the satisfaction rating.
     - **Kritik & Saran:** Textarea for additional feedback.
  4. Implement form state management using `useState`:
     ```typescript
     const [form, setForm] = useState({ name: '', email: '', rating: '', feedback: '' });
     const [loading, setLoading] = useState(false);
     const [message, setMessage] = useState('');
     ```
  5. Create an `onSubmit` function to validate and send data:
     ```typescript
     const handleSubmit = async (e: React.FormEvent) => {
       e.preventDefault();
       if (!form.name || !form.email || !form.rating) {
         setMessage('Mohon isi field yang wajib.');
         return;
       }
       setLoading(true);
       try {
         const res = await fetch('/api/survey', {
           method: 'POST',
           headers: { 'Content-Type': 'application/json' },
           body: JSON.stringify(form)
         });
         const data = await res.json();
         if (res.ok) {
           setMessage('Survei berhasil dikirim. Terima kasih!');
           setForm({ name: '', email: '', rating: '', feedback: '' });
         } else {
           setMessage(data.error || 'Terjadi kesalahan.');
         }
       } catch (err) {
         setMessage('Terjadi kesalahan saat mengirim survei.');
       }
       setLoading(false);
     };
     ```
  6. Design the UI with modern styling using Tailwind CSS (or existing classes):
     ```jsx
     export default function SurveyPage() {
       return (
         <div className="max-w-xl mx-auto p-6">
           <h1 className="text-3xl font-bold mb-4 text-center">Survei Kepuasan Masyarakat</h1>
           <form onSubmit={handleSubmit} className="space-y-4">
             <div>
               <label className="block mb-1 font-medium">Nama</label>
               <Input
                 type="text"
                 value={form.name}
                 onChange={(e) => setForm({ ...form, name: e.target.value })}
                 placeholder="Masukkan nama Anda"
                 className="w-full"
               />
             </div>
             <div>
               <label className="block mb-1 font-medium">Email</label>
               <Input
                 type="email"
                 value={form.email}
                 onChange={(e) => setForm({ ...form, email: e.target.value })}
                 placeholder="Masukkan email Anda"
                 className="w-full"
               />
             </div>
             <div>
               <label className="block mb-1 font-medium">Rating Kepuasan (1-5)</label>
               <input
                 type="number"
                 min="1"
                 max="5"
                 value={form.rating}
                 onChange={(e) => setForm({ ...form, rating: e.target.value })}
                 placeholder="1 = sangat tidak puas, 5 = sangat puas"
                 className="border rounded px-3 py-2 w-full"
               />
             </div>
             <div>
               <label className="block mb-1 font-medium">Kritik & Saran</label>
               <Textarea
                 value={form.feedback}
                 onChange={(e) => setForm({ ...form, feedback: e.target.value })}
                 placeholder="Tuliskan kritik dan saran Anda"
                 className="w-full"
               />
             </div>
             {message && <p className="text-center text-sm text-red-600">{message}</p>}
             <div className="text-center">
               <Button type="submit" disabled={loading}>
                 {loading ? 'Mengirim...' : 'Kirim Survei'}
               </Button>
             </div>
           </form>
         </div>
       );
     }
     ```
  7. **UI/UX Considerations:**  
     • The layout is centered and uses clear typography ensuring readability.  
     • Input fields have consistent spacing and modern styling without using external icon libraries.

## 5. Navigation (Optional)
- **File:** `src/app/layout.tsx` (if it exists)  
- **Changes / Implementation Steps:**
  1. Add a navigation link to the survey page:
     ```jsx
     <nav className="p-4 bg-gray-100">
       <a href="/survey" className="text-blue-600 hover:underline">
         Survei Kepuasan Masyarakat
       </a>
     </nav>
     ```
  2. Ensure the link is styled to fit the modern design of the application.

## 6. Testing and Error Handling
- **API Testing:**  
  Use the following curl command to test the API endpoint:
  ```bash
  curl -X POST http://localhost:3000/api/survey \
    -H "Content-Type: application/json" \
    -d '{"name": "Budi", "email": "budi@example.com", "rating": "5", "feedback": "Layanan sangat memuaskan"}'
  ```
- **Client Testing:**  
  • Run the Next.js app locally (e.g., using `npm run dev`).  
  • Verify that form validation, loading states, and success/error messages are correctly displayed.

## Summary
- Created a new `.env.local` to securely store Google Sheets credentials.  
- Implemented `src/lib/googleSheets.ts` for connecting to the Google Sheets API and appending data.  
- Developed an API endpoint in `src/app/api/survey/route.ts` with validation and error handling.  
- Built a modern, responsive survey form in `src/app/survey/page.tsx` using shadcn UI components and Tailwind CSS.  
- Optionally updated navigation for easy access to the survey page.  
- Verified the end-to-end flow using curl and browser tests, ensuring robust error handling and clean UI/UX.
