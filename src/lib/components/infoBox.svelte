<script lang="ts">
  import DANGERLogo from '$lib/img/DANGER_logo.svg';
  import ERCLogo from '$lib/img/ERC_logo.svg';
  import UniLogo from '$lib/img/Universität_Witten-Herdecke.svg';
  let showModal = $state(false);
  let {showSuggestions = $bindable(),
        countryOpen = $bindable(),
        filterOpen = $bindable()} = $props()

  function openFullText() {
    showModal = true;
    showSuggestions = false;
    countryOpen = false;
    filterOpen = false;

    // Prevent scrolling when modal is open
    document.body.style.overflow = 'hidden';
  }

  function closeModal() {
    showModal = false;
    // Restore scrolling when modal closes
    document.body.style.overflow = 'auto';
  }

</script>

<!-- Info Button -->
<button class="info-button" onclick={openFullText} aria-label="Show dataset information">
  <svg xmlns="http://www.w3.org/2000/svg" width="40" height="40" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
    <circle cx="12" cy="12" r="10"></circle>
    <line x1="12" y1="16" x2="12" y2="12"></line>
    <line x1="12" y1="8" x2="12.01" y2="8"></line>
  </svg>
</button>

<!-- Modal -->
{#if showModal}
    <div 
      class="modal" 
      role="dialog"
      aria-modal="true"
      aria-labelledby="modal-title"
    >
      <div class="modal-content">
        <div class="modal-header">
          <h2 id="modal-title">How to use the map</h2>
          <button 
            type="button"
            class="close-btn" 
            onclick={closeModal}
            aria-label="Close modal"
          >
            ×
          </button>
        </div>
        
        <div class="modal-body">
          <div class="content-section">
            <p>Each point on the map represents a news report of a political violence event. Multiple reports may describe the same event. Zoom in to see individual reports more clearly, outlined in black. Select single events to see the date and actors involved. Use the timeline slider to select a period of interest. When you select an actor group or country, the timeline shows the share of records associated with that selection. Toggle to single-country view to restrict the timeline to the selected country.</p> <br>
            <h3>About the CAIN dataset</h3>
            <p>The Citizen Anger Interwar News (CAIN) dataset records reports of political violence in Europe's interwar democracies (1919-1939). Political violence includes any event that saw the use of force that resulted in at least one injury by a government or a non-state actor. Violence could have been one-sided (against civilians) or reciprocal. The data is organized at the level of news reports and several news reports might describe the same event.</p>
          </div>

          <div class="content-section">

            <p>The current release of the data contains violent events for eight European democracies. We plan to add more countries in the future. The CAIN data does not extend to authoritarian regime periods, such as Germany after March 1933. Country borders depicted in the map reflect the European borders in early 1926. In future releases, we aim to implement dynamic border changes. Country borders were sourced from the CShapes dataset:</p>
          <br>
            <p class="citation">Schvitz, Guy, Seraina Rüegger, Luc Girardin, Lars-Erik Cederman, Nils Weidmann, and Kristian Skrede Gleditsch. 2022. "Mapping The International System, 1886-2017: The CShapes 2.0 Dataset." <em>Journal of Conflict Resolution</em> 66(1): 144–61.</p>

          </div>

          <div class="content-section">
            <p>This project was supported by the European Research Council Starting Grant "Democracy, Anger, and Elite Responses" (DANGER), Project No. 950359.</p>
          </div>

          <div class="logo-section">
              <div class="logos-container">
                  <img src={DANGERLogo} alt="DANGER Project Logo" class="logo" />
                  <img src={ERCLogo} alt="European Research Council Logo" class="logo" />
                  <img src={UniLogo} alt="University of Witten/Herdecke Logo" class="logo" />
              </div>
          </div>
        </div>
      </div>
    </div>
{/if}

<style>
  /* Info Button Styles */
  .info-button {
    position: absolute;
    bottom: 20px;
    right: 20px;
    width: 50px;
    height: 50px;
    border-radius: 50%;
    background-color: #555;
    color: white;
    border: none;
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
    transition: all 0.2s ease;
  }

  .info-button:hover {
    background-color: #fec604;
    transform: scale(1.05);
    box-shadow: 0 6px 16px rgba(0, 0, 0, 0.2);
  }

  .info-button:active {
    transform: scale(0.95);
  }

  .modal {
    position: fixed !important;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-color: rgba(0, 0, 0, 0.5);
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 1000;
    animation: fadeIn 0.3s ease;
    overflow-y: auto;
  }

  .modal-content {
    background-color: white;
    border-radius: 12px;
    box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
    max-height: 90vh;
    display: flex;
    flex-direction: column;
    animation: slideUp 0.3s ease;
    overflow: hidden;
  }

  /* Mobile-first */
  .modal-content {
    width: 90%;
    /* max-width: 500px; */
    height: 90%;
    max-height: 90vh;
    border-radius: 12px;
    margin: auto;
}

  /* Desktop  */
  @media (min-width: 768px) {
    .modal-content {
      width: 50%;
      min-width: 60%;
      max-width: 80%;
      height: auto;
      max-height: 80vh;
      border-radius: 12px;
    }
  }

  /* Modal Header */
  .modal-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 24px;
    border-bottom: 1px solid #e9ecef;
    background-color: #f8f9fa;
  }

  #modal-title {
    margin: 0;
    font-size: 1.5rem;
    font-weight: 600;
    color: #333;
  }

  /* Close Button */
  .close-btn {
    background: none;
    border: none;
    font-size: 32px;
    color: #666;
    cursor: pointer;
    width: 40px;
    height: 40px;
    display: flex;
    align-items: center;
    justify-content: center;
    border-radius: 50%;
    transition: all 0.2s ease;
    line-height: 1;
    padding: 0;
  }

  .close-btn:hover {
    background-color: #f0f0f0;
    color: #333;
  }

  /* Modal Body */
  .modal-body {
    padding: 24px;
    overflow-y: auto;
    flex-grow: 1;
  }

  .content-section {
    margin-bottom: 28px;
  }

  .content-section:last-child {
    margin-bottom: 0;
  }

  .citation {
    font-style: italic;
    font-size: 0.9rem;
    color: #555;
    border-left: 3px solid #fdae2a;
    padding-left: 1rem;
    margin-top: 1.5rem;
    }

.citation em {
  font-style: normal;
  font-weight: 500;
}

  .content-section p {
    margin: 0;
    line-height: 1.6;
    color: #555;
    font-size: 1rem;
  }

  .logo-section {
  margin-top: 2rem;
  padding-top: 1.5rem;
  border-top: 1px solid #ddd;
}

.logos-container {
  display: flex;
  justify-content: center;
  align-items: center;
  flex-wrap: wrap;
  gap: 2rem;
}

.logo {
  height: 50px;
  max-width: 150px;
  object-fit: contain;
  opacity: 0.8;
  transition: all 0.15s ease;
}



/* For mobile responsiveness */
@media (max-width: 640px) {
  .logos-container {
    gap: 1.5rem;
  }
  
  .logo {
    height: 40px;
    max-width: 120px;
  }
}

  /* Animations */
  @keyframes fadeIn {
    from {
      opacity: 0;
    }
    to {
      opacity: 1;
    }
  }

  @keyframes slideUp {
    from {
      transform: translateY(20px);
      opacity: 0;
    }
    to {
      transform: translateY(0);
      opacity: 1;
    }
  }

  /* Scrollbar styling */
  .modal-body::-webkit-scrollbar {
    width: 8px;
  }

  .modal-body::-webkit-scrollbar-track {
    background: #f1f1f1;
    border-radius: 4px;
  }

  .modal-body::-webkit-scrollbar-thumb {
    background: #c1c1c1;
    border-radius: 4px;
  }

  .modal-body::-webkit-scrollbar-thumb:hover {
    background: #a8a8a8;
  }
</style>